---
title: "Firebase Realtime Database完全ガイド — リアルタイム同期/オフライン/Compose連携"
emoji: "🔥"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Firebase Realtime Database**（リアルタイム同期、オフライン対応、CRUD操作、セキュリティルール、Compose連携）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
    implementation("com.google.firebase:firebase-database-ktx")
}
```

---

## データモデル

```kotlin
data class ChatMessage(
    val id: String = "",
    val userId: String = "",
    val userName: String = "",
    val text: String = "",
    val timestamp: Long = System.currentTimeMillis()
)
```

---

## Repository

```kotlin
class ChatRepository @Inject constructor() {
    private val database = Firebase.database
    private val messagesRef = database.getReference("messages")

    init {
        database.setPersistenceEnabled(true) // オフライン対応
    }

    fun getMessages(): Flow<List<ChatMessage>> = callbackFlow {
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) {
                val messages = snapshot.children.mapNotNull {
                    it.getValue(ChatMessage::class.java)?.copy(id = it.key ?: "")
                }.sortedBy { it.timestamp }
                trySend(messages)
            }

            override fun onCancelled(error: DatabaseError) {
                close(error.toException())
            }
        }

        messagesRef.orderByChild("timestamp").limitToLast(100).addValueEventListener(listener)
        awaitClose { messagesRef.removeEventListener(listener) }
    }

    suspend fun sendMessage(message: ChatMessage) {
        val key = messagesRef.push().key ?: return
        messagesRef.child(key).setValue(message.copy(id = key)).await()
    }

    suspend fun deleteMessage(messageId: String) {
        messagesRef.child(messageId).removeValue().await()
    }
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class ChatViewModel @Inject constructor(
    private val repository: ChatRepository
) : ViewModel() {

    val messages = repository.getMessages()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun sendMessage(text: String, userId: String, userName: String) {
        viewModelScope.launch {
            repository.sendMessage(
                ChatMessage(userId = userId, userName = userName, text = text)
            )
        }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel = hiltViewModel()) {
    val messages by viewModel.messages.collectAsStateWithLifecycle()
    var inputText by remember { mutableStateOf("") }

    Column(Modifier.fillMaxSize()) {
        LazyColumn(
            Modifier.weight(1f).padding(horizontal = 16.dp),
            reverseLayout = true
        ) {
            items(messages.reversed(), key = { it.id }) { message ->
                MessageBubble(message)
            }
        }

        Row(Modifier.fillMaxWidth().padding(8.dp), verticalAlignment = Alignment.CenterVertically) {
            OutlinedTextField(
                value = inputText,
                onValueChange = { inputText = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("メッセージを入力") }
            )
            IconButton(onClick = {
                if (inputText.isNotBlank()) {
                    viewModel.sendMessage(inputText, "user-1", "ユーザー")
                    inputText = ""
                }
            }) {
                Icon(Icons.AutoMirrored.Filled.Send, "送信")
            }
        }
    }
}

@Composable
fun MessageBubble(message: ChatMessage) {
    Column(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
        Text(message.userName, style = MaterialTheme.typography.labelSmall)
        Card { Text(message.text, Modifier.padding(8.dp)) }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| リアルタイム同期 | `addValueEventListener` |
| オフライン | `setPersistenceEnabled(true)` |
| 書き込み | `push().setValue()` |
| クエリ | `orderByChild().limitToLast()` |
| Flow変換 | `callbackFlow` |

- `callbackFlow`でリアルタイムリスナーをFlow化
- `setPersistenceEnabled`でオフライン自動キャッシュ
- `push()`で一意キー自動生成
- セキュリティルールで読み書き権限制御

---

8種類のAndroidアプリテンプレート（Firebase連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Firestore](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-firestore-2026)
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
