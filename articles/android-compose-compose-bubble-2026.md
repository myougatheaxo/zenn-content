---
title: "Compose Bubble完全ガイド — バブル通知/フローティングUI/チャットバブル"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "notification"]
published: true
---

## この記事で学べること

**Compose Bubble**（バブル通知、フローティングチャット、BubbleMetadata、バブルActivity）を解説します。

---

## バブル通知設定

```kotlin
// AndroidManifest.xml
// <activity android:name=".BubbleActivity"
//     android:resizeableActivity="true"
//     android:allowEmbedded="true" />

class NotificationHelper(private val context: Context) {
    private val channelId = "chat_channel"

    fun createChannel() {
        val channel = NotificationChannel(channelId, "チャット",
            NotificationManager.IMPORTANCE_HIGH)
        context.getSystemService(NotificationManager::class.java)
            .createNotificationChannel(channel)
    }

    fun showBubble(sender: String, message: String) {
        val bubbleIntent = PendingIntent.getActivity(
            context, 0,
            Intent(context, BubbleActivity::class.java).apply {
                putExtra("sender", sender)
            },
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
        )

        val bubbleMetadata = Notification.BubbleMetadata.Builder(
            bubbleIntent,
            Icon.createWithResource(context, R.drawable.ic_chat)
        ).setDesiredHeight(600)
            .setAutoExpandBubble(false)
            .setSuppressNotification(false)
            .build()

        val person = Person.Builder().setName(sender)
            .setIcon(Icon.createWithResource(context, R.drawable.ic_person))
            .build()

        val notification = Notification.Builder(context, channelId)
            .setSmallIcon(R.drawable.ic_chat)
            .setBubbleMetadata(bubbleMetadata)
            .setCategory(Notification.CATEGORY_MESSAGE)
            .setStyle(Notification.MessagingStyle(person)
                .addMessage(message, System.currentTimeMillis(), person))
            .addPerson(person)
            .build()

        context.getSystemService(NotificationManager::class.java)
            .notify(sender.hashCode(), notification)
    }
}
```

---

## バブルActivity（Compose）

```kotlin
class BubbleActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val sender = intent.getStringExtra("sender") ?: "不明"
        setContent {
            MaterialTheme {
                BubbleChatScreen(sender)
            }
        }
    }
}

@Composable
fun BubbleChatScreen(sender: String) {
    var message by remember { mutableStateOf("") }
    val messages = remember { mutableStateListOf("こんにちは！", "元気ですか？") }

    Column(Modifier.fillMaxSize()) {
        // ヘッダー
        TopAppBar(title = { Text(sender) })

        // メッセージ一覧
        LazyColumn(Modifier.weight(1f).padding(8.dp),
            reverseLayout = true) {
            items(messages.reversed()) { msg ->
                Card(Modifier.padding(4.dp)) {
                    Text(msg, Modifier.padding(12.dp))
                }
            }
        }

        // 入力欄
        Row(Modifier.padding(8.dp), verticalAlignment = Alignment.CenterVertically) {
            OutlinedTextField(
                value = message, onValueChange = { message = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("メッセージ") }
            )
            IconButton(onClick = {
                if (message.isNotBlank()) {
                    messages.add(message); message = ""
                }
            }) { Icon(Icons.Default.Send, "送信") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BubbleMetadata` | バブル設定 |
| `MessagingStyle` | チャット通知 |
| `Person` | 送信者情報 |
| `allowEmbedded` | バブル埋め込み許可 |

- `BubbleMetadata`で通知をバブル表示
- `resizeableActivity`+`allowEmbedded`がManifestに必要
- `MessagingStyle`でチャット形式の通知
- バブルActivityはComposeで自由にUI構築

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FirebaseMessaging](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-messaging-2026)
- [Compose ForegroundService](https://zenn.dev/myougatheaxo/articles/android-compose-compose-foreground-service-2026)
- [Compose Shortcuts](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shortcuts-2026)
