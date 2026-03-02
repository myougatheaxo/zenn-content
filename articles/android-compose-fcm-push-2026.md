---
title: "FCMプッシュ通知 + Compose連携ガイド"
emoji: "📲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Firebase Cloud Messaging (FCM)**でプッシュ通知を実装し、Composeアプリと連携する方法を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.0.0"))
    implementation("com.google.firebase:firebase-messaging-ktx")
}
```

---

## MessagingService

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        // トークンをサーバーに送信
        sendTokenToServer(token)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        message.notification?.let { notification ->
            showNotification(
                title = notification.title ?: "通知",
                body = notification.body ?: ""
            )
        }

        // データペイロード処理
        message.data.isNotEmpty().let {
            val type = message.data["type"]
            val payload = message.data["payload"]
            handleDataMessage(type, payload)
        }
    }

    private fun showNotification(title: String, body: String) {
        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent, PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(this, "fcm_channel")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(this).notify(
            System.currentTimeMillis().toInt(),
            notification
        )
    }
}
```

---

## AndroidManifest設定

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

## トークン取得とCompose連携

```kotlin
@HiltViewModel
class NotificationViewModel @Inject constructor() : ViewModel() {

    private val _fcmToken = MutableStateFlow<String?>(null)
    val fcmToken = _fcmToken.asStateFlow()

    init {
        fetchToken()
    }

    private fun fetchToken() {
        FirebaseMessaging.getInstance().token
            .addOnSuccessListener { token ->
                _fcmToken.value = token
            }
    }

    fun subscribeToTopic(topic: String) {
        FirebaseMessaging.getInstance().subscribeToTopic(topic)
    }

    fun unsubscribeFromTopic(topic: String) {
        FirebaseMessaging.getInstance().unsubscribeFromTopic(topic)
    }
}

@Composable
fun NotificationSettingsScreen(viewModel: NotificationViewModel = hiltViewModel()) {
    var newsEnabled by remember { mutableStateOf(true) }
    var updatesEnabled by remember { mutableStateOf(true) }

    LazyColumn(Modifier.padding(16.dp)) {
        item {
            SwitchPreference(
                title = "ニュース通知",
                checked = newsEnabled,
                onCheckedChange = { enabled ->
                    newsEnabled = enabled
                    if (enabled) viewModel.subscribeToTopic("news")
                    else viewModel.unsubscribeFromTopic("news")
                }
            )
        }
        item {
            SwitchPreference(
                title = "アップデート通知",
                checked = updatesEnabled,
                onCheckedChange = { enabled ->
                    updatesEnabled = enabled
                    if (enabled) viewModel.subscribeToTopic("updates")
                    else viewModel.unsubscribeFromTopic("updates")
                }
            )
        }
    }
}
```

---

## まとめ

- `FirebaseMessagingService`でメッセージ受信
- `onNewToken`でトークン更新をサーバーに通知
- `subscribeToTopic`/`unsubscribeFromTopic`でトピック管理
- `NotificationCompat`でAndroid通知表示
- `PendingIntent.FLAG_IMMUTABLE`でセキュリティ対応
- Android 13+は`POST_NOTIFICATIONS`パーミッション必須

---

8種類のAndroidアプリテンプレート（プッシュ通知対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [通知実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-notification-2026)
- [WorkManager連携](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-2026)
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
