---
title: "Firebase Cloud Messaging完全ガイド — プッシュ通知/トピック/データメッセージ"
emoji: "📨"
type: "tech"
topics: ["android", "firebase", "kotlin", "notification"]
published: true
---

## この記事で学べること

**Firebase Cloud Messaging（FCM）**（プッシュ通知受信、トピック購読、データメッセージ処理、トークン管理）を解説します。

---

## FCMサービス

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        // サーバーにトークン送信
        CoroutineScope(Dispatchers.IO).launch {
            sendTokenToServer(token)
        }
    }

    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        // 通知メッセージ
        remoteMessage.notification?.let { notification ->
            showNotification(notification.title ?: "", notification.body ?: "")
        }

        // データメッセージ
        if (remoteMessage.data.isNotEmpty()) {
            handleDataMessage(remoteMessage.data)
        }
    }

    private fun handleDataMessage(data: Map<String, String>) {
        val type = data["type"]
        when (type) {
            "chat" -> showChatNotification(data["sender"] ?: "", data["message"] ?: "")
            "update" -> showUpdateNotification(data["version"] ?: "")
        }
    }

    private fun showNotification(title: String, body: String) {
        val notification = NotificationCompat.Builder(this, "fcm_channel")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(this).notify(System.currentTimeMillis().toInt(), notification)
    }
}
```

---

## トピック購読

```kotlin
class FcmRepository @Inject constructor() {

    fun subscribeToTopic(topic: String): Task<Void> {
        return FirebaseMessaging.getInstance().subscribeToTopic(topic)
    }

    fun unsubscribeFromTopic(topic: String): Task<Void> {
        return FirebaseMessaging.getInstance().unsubscribeFromTopic(topic)
    }

    suspend fun getToken(): String? {
        return try {
            FirebaseMessaging.getInstance().token.await()
        } catch (e: Exception) {
            null
        }
    }
}

// Compose UI
@Composable
fun TopicSubscriptionUI(viewModel: FcmViewModel = hiltViewModel()) {
    val topics = listOf("news", "updates", "promotions")
    val subscribed by viewModel.subscribedTopics.collectAsStateWithLifecycle(emptySet())

    Column(Modifier.padding(16.dp)) {
        Text("通知設定", style = MaterialTheme.typography.titleLarge)
        topics.forEach { topic ->
            Row(
                Modifier.fillMaxWidth().padding(vertical = 8.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(topic)
                Switch(
                    checked = topic in subscribed,
                    onCheckedChange = { checked ->
                        if (checked) viewModel.subscribe(topic)
                        else viewModel.unsubscribe(topic)
                    }
                )
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| トークン管理 | `onNewToken` |
| 通知受信 | `onMessageReceived` |
| トピック | `subscribeToTopic` |
| データメッセージ | `remoteMessage.data` |

- `FirebaseMessagingService`でプッシュ通知受信
- トピック購読でセグメント配信
- データメッセージでカスタム処理
- `onNewToken`でトークン更新を管理

---

8種類のAndroidアプリテンプレート（プッシュ通知対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ローカル通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-2026)
- [Firebase Crashlytics](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-crashlytics-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
