---
title: "SSE/WebSocket完全ガイド — リアルタイム通信/OkHttp/Flow統合"
emoji: "🔌"
type: "tech"
topics: ["android", "kotlin", "websocket", "network"]
published: true
---

## この記事で学べること

**SSE/WebSocket**（OkHttp WebSocket、Server-Sent Events、Flow統合、再接続、Compose UI）を解説します。

---

## WebSocket (OkHttp)

```kotlin
class WebSocketClient @Inject constructor(
    private val okHttpClient: OkHttpClient
) {
    fun connect(url: String): Flow<WebSocketEvent> = callbackFlow {
        val request = Request.Builder().url(url).build()

        val webSocket = okHttpClient.newWebSocket(request, object : WebSocketListener() {
            override fun onOpen(webSocket: WebSocket, response: Response) {
                trySend(WebSocketEvent.Connected)
            }

            override fun onMessage(webSocket: WebSocket, text: String) {
                trySend(WebSocketEvent.Message(text))
            }

            override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
                trySend(WebSocketEvent.Closing(code, reason))
                webSocket.close(code, reason)
            }

            override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
                trySend(WebSocketEvent.Error(t))
                close(t)
            }
        })

        awaitClose { webSocket.close(1000, "Client closed") }
    }
}

sealed interface WebSocketEvent {
    data object Connected : WebSocketEvent
    data class Message(val text: String) : WebSocketEvent
    data class Closing(val code: Int, val reason: String) : WebSocketEvent
    data class Error(val throwable: Throwable) : WebSocketEvent
}
```

---

## 自動再接続

```kotlin
fun connectWithRetry(url: String): Flow<WebSocketEvent> {
    return connect(url)
        .retryWhen { cause, attempt ->
            if (attempt < 5) {
                val delayMs = minOf(1000L * (1 shl attempt.toInt()), 30_000L) // 指数バックオフ
                delay(delayMs)
                emit(WebSocketEvent.Reconnecting(attempt + 1))
                true
            } else false
        }
}
```

---

## Server-Sent Events (SSE)

```kotlin
class SseClient @Inject constructor(
    private val okHttpClient: OkHttpClient
) {
    fun connect(url: String): Flow<SseEvent> = callbackFlow {
        val request = Request.Builder()
            .url(url)
            .header("Accept", "text/event-stream")
            .build()

        val call = okHttpClient.newCall(request)

        call.enqueue(object : Callback {
            override fun onResponse(call: Call, response: Response) {
                response.body?.source()?.let { source ->
                    var eventType = ""
                    val dataBuilder = StringBuilder()

                    while (!source.exhausted()) {
                        val line = source.readUtf8Line() ?: break

                        when {
                            line.startsWith("event:") -> eventType = line.removePrefix("event:").trim()
                            line.startsWith("data:") -> dataBuilder.append(line.removePrefix("data:").trim())
                            line.isEmpty() -> {
                                trySend(SseEvent(eventType, dataBuilder.toString()))
                                eventType = ""
                                dataBuilder.clear()
                            }
                        }
                    }
                }
            }

            override fun onFailure(call: Call, e: IOException) {
                close(e)
            }
        })

        awaitClose { call.cancel() }
    }
}

data class SseEvent(val type: String, val data: String)
```

---

## Compose UI

```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel = hiltViewModel()) {
    val messages by viewModel.messages.collectAsStateWithLifecycle()
    val isConnected by viewModel.isConnected.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize()) {
        // 接続状態バー
        AnimatedVisibility(!isConnected) {
            Surface(color = MaterialTheme.colorScheme.errorContainer, modifier = Modifier.fillMaxWidth()) {
                Text("接続中...", Modifier.padding(8.dp))
            }
        }

        // メッセージ一覧
        LazyColumn(Modifier.weight(1f), reverseLayout = true) {
            items(messages) { message ->
                MessageBubble(message)
            }
        }

        // 入力欄
        var input by remember { mutableStateOf("") }
        Row(Modifier.padding(8.dp)) {
            OutlinedTextField(
                value = input, onValueChange = { input = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("メッセージ") }
            )
            IconButton(onClick = { viewModel.send(input); input = "" }) {
                Icon(Icons.AutoMirrored.Filled.Send, "送信")
            }
        }
    }
}
```

---

## まとめ

| 技術 | 用途 | 方向 |
|------|------|------|
| WebSocket | チャット/ゲーム | 双方向 |
| SSE | 通知/フィード | サーバー→クライアント |
| Polling | 単純な更新確認 | クライアント→サーバー |

- `callbackFlow`でWebSocket/SSEをFlow化
- 指数バックオフで自動再接続
- SSEはHTTPベースで実装がシンプル
- WebSocketは双方向リアルタイム通信に最適

---

8種類のAndroidアプリテンプレート（リアルタイム通信対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [Flow/StateFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-stateflow-patterns-2026)
- [ネットワーク監視](https://zenn.dev/myougatheaxo/articles/android-compose-connectivity-manager-2026)
