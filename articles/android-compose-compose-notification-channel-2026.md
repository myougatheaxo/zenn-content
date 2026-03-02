---
title: "Compose NotificationChannel完全ガイド — チャネル管理/グループ/重要度設定"
emoji: "🔔"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "notification"]
published: true
---

## この記事で学べること

**Compose NotificationChannel**（通知チャネル作成、グループ分け、重要度設定、カスタム通知）を解説します。

---

## チャネル作成

```kotlin
object NotificationChannels {
    const val MESSAGES = "messages"
    const val UPDATES = "updates"
    const val PROMOTIONS = "promotions"

    fun createAll(context: Context) {
        val manager = context.getSystemService(NotificationManager::class.java)

        // グループ
        manager.createNotificationChannelGroup(
            NotificationChannelGroup("social", "ソーシャル")
        )

        val channels = listOf(
            NotificationChannel(MESSAGES, "メッセージ", NotificationManager.IMPORTANCE_HIGH).apply {
                description = "新着メッセージの通知"
                group = "social"
                enableVibration(true)
                setShowBadge(true)
            },
            NotificationChannel(UPDATES, "アップデート", NotificationManager.IMPORTANCE_DEFAULT).apply {
                description = "アプリの更新情報"
            },
            NotificationChannel(PROMOTIONS, "プロモーション", NotificationManager.IMPORTANCE_LOW).apply {
                description = "セール・キャンペーン情報"
                setShowBadge(false)
            }
        )
        manager.createNotificationChannels(channels)
    }
}

// Application.onCreate()で呼び出し
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        NotificationChannels.createAll(this)
    }
}
```

---

## 通知送信

```kotlin
fun sendNotification(context: Context, channelId: String, title: String, body: String) {
    val intent = PendingIntent.getActivity(context, 0,
        Intent(context, MainActivity::class.java),
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)

    val notification = NotificationCompat.Builder(context, channelId)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(body)
        .setContentIntent(intent)
        .setAutoCancel(true)
        .build()

    if (ActivityCompat.checkSelfPermission(context,
            Manifest.permission.POST_NOTIFICATIONS) == PackageManager.PERMISSION_GRANTED) {
        NotificationManagerCompat.from(context).notify(System.currentTimeMillis().toInt(), notification)
    }
}

@Composable
fun NotificationDemoScreen() {
    val context = LocalContext.current
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { /* granted */ }

    LaunchedEffect(Unit) {
        if (Build.VERSION.SDK_INT >= 33) {
            launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Button(onClick = {
            sendNotification(context, NotificationChannels.MESSAGES, "新着メッセージ", "田中さんからメッセージ")
        }) { Text("メッセージ通知") }

        Button(onClick = {
            sendNotification(context, NotificationChannels.UPDATES, "アップデート", "v2.0が利用可能です")
        }) { Text("アップデート通知") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `NotificationChannel` | 通知チャネル |
| `NotificationChannelGroup` | チャネルグループ |
| `IMPORTANCE_*` | 重要度設定 |
| `POST_NOTIFICATIONS` | Android 13+権限 |

- Android 8.0+ではNotificationChannel必須
- 重要度でユーザーへの表示方法が変わる
- `NotificationChannelGroup`で関連チャネルをグループ化
- Android 13+では`POST_NOTIFICATIONS`パーミッション必要

---

8種類のAndroidアプリテンプレート（通知対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FirebaseMessaging](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-messaging-2026)
- [Compose Bubble](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bubble-2026)
- [Compose ForegroundService](https://zenn.dev/myougatheaxo/articles/android-compose-compose-foreground-service-2026)
