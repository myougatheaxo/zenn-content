---
title: "Ktor WebSocket完全ガイド — リアルタイム通信/チャット/再接続"
emoji: "🔌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ktor"]
published: true
---

## この記事で学べること

**Ktor WebSocket**（WebSocket接続、リアルタイムチャット、自動再接続、Compose統合）を解説します。

---

## 基本WebSocket

```kotlin
// build.gradle
// implementation "io.ktor:ktor-client-websockets:2.3.7"

class WebSocketClient {
    private val client = HttpClient(CIO) {
        install(WebSockets)
    }

    suspend fun connect(
        url: String,
        onMessage: (String) -> Unit,
        onClose: () -> Unit
    ) {
        client.webSocket(url) {
            for (frame in incoming) {
                when (frame) {
                    is Frame.Text -> onMessage(frame.readText())
                    is Frame.Close -> { onClose(); break }
                    else -> {}
                }
            }
        }
    }

    fun close() {
        client.close()
    }
}
```

---

## チャットViewModel

```kotlin
@HiltViewModel
class ChatViewModel @Inject constructor() : ViewModel() {
    private val _messages = MutableStateFlow<List<ChatMessage>>(emptyList())
    val messages: StateFlow<List<ChatMessage>> = _messages.asStateFlow()

    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState.asStateFlow()

    private var session: DefaultWebSocketSession? = null
    private val client = HttpClient(CIO) { install(WebSockets) }

    fun connect(url: String) {
        viewModelScope.launch {
            _connectionState.value = ConnectionState.CONNECTING
            try {
                client.webSocket(url) {
                    session = this
                    _connectionState.value = ConnectionState.CONNECTED

                    for (frame in incoming) {
                        if (frame is Frame.Text) {
                            val msg = Json.decodeFromString<ChatMessage>(frame.readText())
                            _messages.update { it + msg }
                        }
                    }
                }
            } catch (e: Exception) {
                _connectionState.value = ConnectionState.DISCONNECTED
                delay(3000)
                connect(url) // 自動再接続
            }
        }
    }

    fun sendMessage(text: String) {
        viewModelScope.launch {
            session?.send(Frame.Text(Json.encodeToString(ChatMessage(text = text))))
        }
    }
}

enum class ConnectionState { CONNECTING, CONNECTED, DISCONNECTED }
```

---

## Compose チャットUI

```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel = hiltViewModel()) {
    val messages by viewModel.messages.collectAsStateWithLifecycle()
    val connectionState by viewModel.connectionState.collectAsStateWithLifecycle()
    var input by remember { mutableStateOf("") }

    LaunchedEffect(Unit) { viewModel.connect("wss://example.com/chat") }

    Column(Modifier.fillMaxSize()) {
        // 接続状態
        if (connectionState != ConnectionState.CONNECTED) {
            Text(
                if (connectionState == ConnectionState.CONNECTING) "接続中..." else "切断",
                Modifier.fillMaxWidth().background(Color.Red.copy(alpha = 0.1f)).padding(8.dp),
                textAlign = TextAlign.Center
            )
        }

        // メッセージ一覧
        LazyColumn(Modifier.weight(1f).padding(8.dp), reverseLayout = true) {
            items(messages.reversed()) { msg ->
                Text(msg.text, Modifier.padding(4.dp))
            }
        }

        // 入力欄
        Row(Modifier.padding(8.dp)) {
            OutlinedTextField(value = input, onValueChange = { input = it }, Modifier.weight(1f), placeholder = { Text("メッセージ") })
            IconButton(onClick = { viewModel.sendMessage(input); input = "" }, enabled = input.isNotBlank()) {
                Icon(Icons.Default.Send, "送信")
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `WebSockets` | Ktor WebSocketプラグイン |
| `webSocket {}` | 接続+送受信 |
| `Frame.Text` | テキストメッセージ |
| `incoming` | 受信チャネル |

- Ktor WebSocketでリアルタイム双方向通信
- `incoming`チャネルで受信メッセージを処理
- `Frame.Text`/`Frame.Binary`でメッセージ種別を区別
- 切断時の自動再接続ロジックを実装

---

8種類のAndroidアプリテンプレート（リアルタイム通信対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Ktor Client](https://zenn.dev/myougatheaxo/articles/android-compose-ktor-client-2026)
- [Flow SharedFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-shared-flow-2026)
- [Coroutine Actor](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-actor-2026)
