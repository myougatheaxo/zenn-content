---
title: "Compose SSE完全ガイド — Server-Sent Events/OkHttp/ストリーミング/ChatGPT風UI"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "sse"]
published: true
---

## この記事で学べること

**Compose SSE**（Server-Sent Events、OkHttpストリーミング、リアルタイムテキスト表示、ChatGPT風UI）を解説します。

---

## SSEクライアント

```kotlin
class SseClient @Inject constructor() {
    private val client = OkHttpClient.Builder()
        .readTimeout(0, TimeUnit.MILLISECONDS) // タイムアウト無効
        .build()

    fun stream(url: String, body: String? = null): Flow<SseEvent> = callbackFlow {
        val requestBuilder = Request.Builder().url(url)
            .header("Accept", "text/event-stream")

        if (body != null) {
            requestBuilder.post(body.toRequestBody("application/json".toMediaType()))
        }

        val call = client.newCall(requestBuilder.build())
        call.enqueue(object : Callback {
            override fun onResponse(call: Call, response: Response) {
                response.body?.source()?.use { source ->
                    var event = ""
                    var data = StringBuilder()
                    while (!source.exhausted()) {
                        val line = source.readUtf8Line() ?: break
                        when {
                            line.startsWith("event:") -> event = line.removePrefix("event:").trim()
                            line.startsWith("data:") -> data.appendLine(line.removePrefix("data:").trim())
                            line.isEmpty() -> {
                                trySend(SseEvent(event, data.toString().trim()))
                                event = ""; data = StringBuilder()
                            }
                        }
                    }
                }
                close()
            }
            override fun onFailure(call: Call, e: IOException) { close(e) }
        })

        awaitClose { call.cancel() }
    }
}

data class SseEvent(val event: String, val data: String)
```

---

## ChatGPT風ストリーミングUI

```kotlin
@HiltViewModel
class ChatViewModel @Inject constructor(
    private val sseClient: SseClient
) : ViewModel() {
    var streamingText by mutableStateOf("")
        private set
    var isStreaming by mutableStateOf(false)
        private set

    fun sendMessage(prompt: String) {
        viewModelScope.launch {
            isStreaming = true
            streamingText = ""

            val body = Json.encodeToString(mapOf("prompt" to prompt))
            sseClient.stream("https://api.example.com/chat", body)
                .collect { event ->
                    if (event.data == "[DONE]") {
                        isStreaming = false
                    } else {
                        streamingText += event.data
                    }
                }
        }
    }
}

@Composable
fun StreamingChatScreen(viewModel: ChatViewModel = hiltViewModel()) {
    var input by remember { mutableStateOf("") }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        // ストリーミングテキスト表示
        Card(Modifier.weight(1f).fillMaxWidth()) {
            Column(Modifier.padding(16.dp).verticalScroll(rememberScrollState())) {
                Text(viewModel.streamingText, style = MaterialTheme.typography.bodyLarge)
                if (viewModel.isStreaming) {
                    Text("▌", color = MaterialTheme.colorScheme.primary)
                }
            }
        }

        Spacer(Modifier.height(16.dp))

        Row(verticalAlignment = Alignment.CenterVertically) {
            OutlinedTextField(
                value = input, onValueChange = { input = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("メッセージを入力") },
                enabled = !viewModel.isStreaming
            )
            Spacer(Modifier.width(8.dp))
            Button(
                onClick = { viewModel.sendMessage(input); input = "" },
                enabled = input.isNotBlank() && !viewModel.isStreaming
            ) { Text("送信") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| SSE | サーバー→クライアント |
| `callbackFlow` | Flow変換 |
| `text/event-stream` | SSEヘッダー |
| ストリーミング表示 | リアルタイムUI |

- SSEでサーバーからのリアルタイムテキストストリーミング
- `callbackFlow`でOkHttpレスポンスをFlowに変換
- `readTimeout(0)`でタイムアウトなしの長時間接続
- ChatGPT風のタイピングエフェクトを実現

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose WebSocket](https://zenn.dev/myougatheaxo/articles/android-compose-compose-websocket-2026)
- [Compose Retrofit](https://zenn.dev/myougatheaxo/articles/android-compose-compose-retrofit-2026)
- [Flow Buffer](https://zenn.dev/myougatheaxo/articles/android-compose-flow-buffer-2026)
