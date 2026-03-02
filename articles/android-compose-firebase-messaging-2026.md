---
title: "Firebase Cloud Messaging完全ガイド — プッシュ通知実装"
emoji: "🔔"
type: "tech"
topics: ["android", "kotlin", "firebase", "notification"]
published: true
---

## この記事で学べること

**Firebase Cloud Messaging (FCM)**（プッシュ通知、トピック購読、データメッセージ、通知チャネル）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts (project)
plugins {
    id("com.google.gms.google-services") version "4.4.2" apply false
}

// build.gradle.kts (app)
plugins {
    id("com.google.gms.google-services")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
    implementation("com.google.firebase:firebase-messaging-ktx")
    implementation("com.google.firebase:firebase-analytics-ktx")
}
```

---

## FirebaseMessagingService

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        // サーバーにトークン送信
        sendTokenToServer(token)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        // 通知メッセージ
        message.notification?.let { notification ->
            showNotification(
                title = notification.title ?: "通知",
                body = notification.body ?: ""
            )
        }

        // データメッセージ
        if (message.data.isNotEmpty()) {
            handleDataMessage(message.data)
        }
    }

    private fun showNotification(title: String, body: String) {
        val channelId = "default"
        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent, PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(this, channelId)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setAutoCancel(true)
            .setContentIntent(pendingIntent)
            .build()

        val manager = getSystemService(NotificationManager::class.java)
        manager.notify(System.currentTimeMillis().toInt(), notification)
    }

    private fun handleDataMessage(data: Map<String, String>) {
        val type = data["type"]
        when (type) {
            "chat" -> handleChatMessage(data)
            "update" -> handleAppUpdate(data)
        }
    }

    private fun sendTokenToServer(token: String) {
        // Retrofit等でサーバーに送信
        CoroutineScope(Dispatchers.IO).launch {
            try {
                apiService.registerToken(TokenRequest(token))
            } catch (e: Exception) {
                Log.e("FCM", "Token registration failed", e)
            }
        }
    }
}
```

---

## AndroidManifest

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

## トークン取得・トピック購読

```kotlin
class NotificationRepository @Inject constructor() {

    // FCMトークン取得
    suspend fun getToken(): String = suspendCancellableCoroutine { cont ->
        FirebaseMessaging.getInstance().token
            .addOnSuccessListener { token -> cont.resume(token) }
            .addOnFailureListener { e -> cont.resumeWithException(e) }
    }

    // トピック購読
    fun subscribeToTopic(topic: String) {
        FirebaseMessaging.getInstance()
            .subscribeToTopic(topic)
            .addOnCompleteListener { task ->
                if (task.isSuccessful) {
                    Log.d("FCM", "Subscribed to $topic")
                }
            }
    }

    // トピック解除
    fun unsubscribeFromTopic(topic: String) {
        FirebaseMessaging.getInstance()
            .unsubscribeFromTopic(topic)
    }
}

// Compose画面での使用
@Composable
fun NotificationSettingsScreen(
    viewModel: SettingsViewModel = hiltViewModel()
) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        Text("通知設定", style = MaterialTheme.typography.headlineMedium)

        Spacer(Modifier.height(16.dp))

        SwitchRow(
            label = "お知らせ",
            checked = settings.newsEnabled,
            onCheckedChange = { viewModel.toggleNews(it) }
        )

        SwitchRow(
            label = "チャット通知",
            checked = settings.chatEnabled,
            onCheckedChange = { viewModel.toggleChat(it) }
        )

        SwitchRow(
            label = "更新通知",
            checked = settings.updateEnabled,
            onCheckedChange = { viewModel.toggleUpdate(it) }
        )
    }
}

@Composable
private fun SwitchRow(
    label: String,
    checked: Boolean,
    onCheckedChange: (Boolean) -> Unit
) {
    Row(
        Modifier.fillMaxWidth().padding(vertical = 8.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(label)
        Switch(checked = checked, onCheckedChange = onCheckedChange)
    }
}
```

---

## 通知チャネル設定

```kotlin
class NotificationHelper(private val context: Context) {

    fun createChannels() {
        val channels = listOf(
            NotificationChannel(
                "chat", "チャット",
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "チャットメッセージの通知"
                enableVibration(true)
            },
            NotificationChannel(
                "news", "お知らせ",
                NotificationManager.IMPORTANCE_DEFAULT
            ).apply {
                description = "アプリのお知らせ"
            },
            NotificationChannel(
                "update", "更新",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "アプリの更新情報"
            }
        )

        val manager = context.getSystemService(NotificationManager::class.java)
        channels.forEach { manager.createNotificationChannel(it) }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| トークン取得 | `FirebaseMessaging.getInstance().token` |
| メッセージ受信 | `onMessageReceived()` |
| トピック購読 | `subscribeToTopic()` |
| 通知表示 | `NotificationCompat.Builder` |
| チャネル管理 | `NotificationChannel` |

- `onNewToken()`でトークン更新時にサーバー同期
- データメッセージでバックグラウンド処理
- トピック購読でグループ通知
- 通知チャネルでユーザー制御

---

8種類のAndroidアプリテンプレート（プッシュ通知対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ローカル通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-schedule-2026)
- [バックグラウンドタスク](https://zenn.dev/myougatheaxo/articles/android-compose-background-task-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
