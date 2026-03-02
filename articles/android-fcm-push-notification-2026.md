---
title: "FCMプッシュ通知完全ガイド — Firebase Cloud Messagingの実装"
emoji: "🔔"
type: "tech"
topics: ["android", "kotlin", "firebase", "notification"]
published: true
---

## この記事で学べること

**Firebase Cloud Messaging (FCM)**でプッシュ通知を受信・表示する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.5.0"))
    implementation("com.google.firebase:firebase-messaging-ktx")
}
```

---

## FirebaseMessagingService

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        // サーバーにトークンを送信
        sendTokenToServer(token)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        message.notification?.let { notification ->
            showNotification(
                title = notification.title ?: "通知",
                body = notification.body ?: ""
            )
        }

        // データペイロード
        if (message.data.isNotEmpty()) {
            handleDataPayload(message.data)
        }
    }

    private fun showNotification(title: String, body: String) {
        val channelId = "default"

        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(this, channelId)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()

        val manager = getSystemService(NotificationManager::class.java)
        manager.notify(System.currentTimeMillis().toInt(), notification)
    }

    private fun handleDataPayload(data: Map<String, String>) {
        val type = data["type"]
        val itemId = data["item_id"]
        // カスタム処理
    }

    private fun sendTokenToServer(token: String) {
        // API呼び出しでサーバーに送信
    }
}
```

---

## AndroidManifest.xml

```xml
<service
    android:name=".MyFirebaseMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

---

## 通知チャネル（Android 8+）

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }

    private fun createNotificationChannel() {
        val channel = NotificationChannel(
            "default",
            "一般通知",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description = "アプリからの通知"
        }
        val manager = getSystemService(NotificationManager::class.java)
        manager.createNotificationChannel(channel)
    }
}
```

---

## トークン取得

```kotlin
// 現在のトークンを取得
FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
    Log.d("FCM", "Token: $token")
}

// トピック購読
FirebaseMessaging.getInstance().subscribeToTopic("news")
    .addOnSuccessListener { Log.d("FCM", "ニューストピック購読完了") }
```

---

## Android 13+ 通知パーミッション

```kotlin
@Composable
fun RequestNotificationPermission() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        val permissionLauncher = rememberLauncherForActivityResult(
            ActivityResultContracts.RequestPermission()
        ) { granted ->
            if (granted) {
                // 通知許可済み
            }
        }

        LaunchedEffect(Unit) {
            permissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
}
```

---

## まとめ

- `FirebaseMessagingService`で通知受信
- `onNewToken`でトークン管理
- `NotificationChannel`をAndroid 8+で必ず作成
- `POST_NOTIFICATIONS`パーミッション（Android 13+）
- トピック購読で一括配信

---

8種類のAndroidアプリテンプレート（通知機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ローカル通知完全ガイド](https://zenn.dev/myougatheaxo/articles/android-notification-guide-2026)
- [WorkManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-workmanager-guide-2026)
- [Firebase Crashlytics入門](https://zenn.dev/myougatheaxo/articles/android-firebase-crashlytics-2026)
