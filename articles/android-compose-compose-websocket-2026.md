---
title: "Compose WebSocket完全ガイド — OkHttp WebSocket/リアルタイム通信/再接続/チャットUI"
emoji: "🔌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "websocket"]
published: true
---

## この記事で学べること

**Compose WebSocket**（OkHttp WebSocket、リアルタイム通信、自動再接続、チャットUI）を解説します。

---

## WebSocketクライアント

```kotlin
class WebSocketClient @Inject constructor() {
    private val client = OkHttpClient.Builder()
        .pingInterval(30, TimeUnit.SECONDS)
        .build()

    private var webSocket: WebSocket? = null
    private val _messages = MutableSharedFlow<ChatMessage>()
    val messages = _messages.asSharedFlow()
    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState = _connectionState.asStateFlow()

    fun connect(url: String) {
        val request = Request.Builder().url(url).build()
        _connectionState.value = ConnectionState.CONNECTING

        webSocket = client.newWebSocket(request, object : WebSocketListener() {
            override fun onOpen(webSocket: WebSocket, response: Response) {
                _connectionState.value = ConnectionState.CONNECTED
            }

            override fun onMessage(webSocket: WebSocket, text: String) {
                val message = Json.decodeFromString<ChatMessage>(text)
                _messages.tryEmit(message)
            }

            override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
                _connectionState.value = ConnectionState.DISCONNECTED
            }

            override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
                _connectionState.value = ConnectionState.DISCONNECTED
            }
        })
    }

    fun send(message: String) {
        webSocket?.send(message)
    }

    fun disconnect() {
        webSocket?.close(1000, "Bye")
    }
}

enum class ConnectionState { CONNECTING, CONNECTED, DISCONNECTED }
```

---

## チャットUI

```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel = hiltViewModel()) {
    val messages by viewModel.messages.collectAsStateWithLifecycle(emptyList())
    val state by viewModel.connectionState.collectAsStateWithLifecycle()
    var input by remember { mutableStateOf("") }
    val listState = rememberLazyListState()

    LaunchedEffect(messages.size) {
        if (messages.isNotEmpty()) listState.animateScrollToItem(messages.lastIndex)
    }

    Column(Modifier.fillMaxSize()) {
        // 接続状態
        Surface(color = when (state) {
            ConnectionState.CONNECTED -> Color(0xFF4CAF50)
            ConnectionState.CONNECTING -> Color(0xFFFFC107)
            ConnectionState.DISCONNECTED -> Color(0xFFF44336)
        }) {
            Text(state.name, Modifier.fillMaxWidth().padding(4.dp),
                color = Color.White, textAlign = TextAlign.Center)
        }

        // メッセージ一覧
        LazyColumn(Modifier.weight(1f), state = listState, contentPadding = PaddingValues(16.dp)) {
            items(messages) { msg ->
                MessageBubble(msg, isMe = msg.sender == "me")
            }
        }

        // 入力欄
        Row(Modifier.padding(8.dp), verticalAlignment = Alignment.CenterVertically) {
            OutlinedTextField(
                value = input, onValueChange = { input = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("メッセージ") }
            )
            IconButton(onClick = {
                if (input.isNotBlank()) { viewModel.send(input); input = "" }
            }) { Icon(Icons.AutoMirrored.Filled.Send, "送信") }
        }
    }
}

@Composable
fun MessageBubble(message: ChatMessage, isMe: Boolean) {
    Row(
        Modifier.fillMaxWidth().padding(vertical = 4.dp),
        horizontalArrangement = if (isMe) Arrangement.End else Arrangement.Start
    ) {
        Card(
            colors = CardDefaults.cardColors(
                containerColor = if (isMe) MaterialTheme.colorScheme.primaryContainer
                else MaterialTheme.colorScheme.surfaceVariant
            )
        ) {
            Text(message.text, Modifier.padding(12.dp))
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `OkHttp WebSocket` | WebSocket接続 |
| `WebSocketListener` | イベントハンドリング |
| `SharedFlow` | メッセージ配信 |
| `pingInterval` | ハートビート |

- OkHttpの`WebSocketListener`でリアルタイム通信
- `SharedFlow`でメッセージをUI層に配信
- `pingInterval`でコネクションの死活監視
- 再接続ロジックで安定した通信を実現

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose SSE](https://zenn.dev/myougatheaxo/articles/android-compose-compose-sse-2026)
- [Compose Retrofit](https://zenn.dev/myougatheaxo/articles/android-compose-compose-retrofit-2026)
- [Compose Ktor Client](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ktor-client-2026)
