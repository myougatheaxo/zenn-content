---
title: "Compose Firebase Messaging完全ガイド — FCM/プッシュ通知/トピック購読"
emoji: "📨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose Firebase Messaging**（FCMプッシュ通知、トピック購読、通知チャネル、データメッセージ）を解説します。

---

## FirebaseMessagingService

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        // サーバーにトークンを送信
        CoroutineScope(Dispatchers.IO).launch {
            ApiClient.registerToken(token)
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

    private fun showNotification(title: String, body: String) {
        val channelId = "default"
        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(this, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)

        val notification = NotificationCompat.Builder(this, channelId)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(this).notify(System.currentTimeMillis().toInt(), notification)
    }
}
```

---

## トピック購読UI

```kotlin
@Composable
fun NotificationSettingsScreen() {
    var newsEnabled by rememberSaveable { mutableStateOf(false) }
    var updatesEnabled by rememberSaveable { mutableStateOf(false) }

    Column(Modifier.padding(16.dp)) {
        Text("通知設定", style = MaterialTheme.typography.titleLarge)
        Spacer(Modifier.height(16.dp))

        NotificationToggle("ニュース通知", newsEnabled) { enabled ->
            newsEnabled = enabled
            if (enabled) Firebase.messaging.subscribeToTopic("news")
            else Firebase.messaging.unsubscribeFromTopic("news")
        }

        NotificationToggle("アップデート通知", updatesEnabled) { enabled ->
            updatesEnabled = enabled
            if (enabled) Firebase.messaging.subscribeToTopic("updates")
            else Firebase.messaging.unsubscribeFromTopic("updates")
        }
    }
}

@Composable
fun NotificationToggle(label: String, checked: Boolean, onCheckedChange: (Boolean) -> Unit) {
    Row(Modifier.fillMaxWidth().padding(vertical = 8.dp), horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically) {
        Text(label)
        Switch(checked = checked, onCheckedChange = onCheckedChange)
    }
}
```

---

## トークン取得

```kotlin
@Composable
fun FCMTokenDisplay() {
    var token by remember { mutableStateOf<String?>(null) }

    LaunchedEffect(Unit) {
        Firebase.messaging.token.addOnSuccessListener { token = it }
    }

    token?.let {
        SelectionContainer {
            Text("FCM Token: ${it.take(20)}...", style = MaterialTheme.typography.bodySmall)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FirebaseMessagingService` | メッセージ受信 |
| `onNewToken` | トークン更新 |
| `subscribeToTopic` | トピック購読 |
| `Firebase.messaging.token` | トークン取得 |

- `FirebaseMessagingService`でバックグラウンド受信
- トピック購読でグループ通知
- Android 13+は`POST_NOTIFICATIONS`パーミッション必要
- 通知チャネルの作成を忘れずに（Android 8+）

---

8種類のAndroidアプリテンプレート（プッシュ通知対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-auth-2026)
- [Firebase Firestore](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-firestore-2026)
- [Compose Notification](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
