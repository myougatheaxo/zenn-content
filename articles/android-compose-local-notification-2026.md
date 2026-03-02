---
title: "ローカル通知完全ガイド — NotificationCompat/チャネル/アクション/スケジュール"
emoji: "🔔"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "notification"]
published: true
---

## この記事で学べること

**ローカル通知**（NotificationCompat、チャネル作成、アクションボタン、スケジュール通知）を解説します。

---

## 通知チャネル作成

```kotlin
class NotificationHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    companion object {
        const val CHANNEL_GENERAL = "general"
        const val CHANNEL_REMINDER = "reminder"
    }

    init {
        createChannels()
    }

    private fun createChannels() {
        val channels = listOf(
            NotificationChannel(CHANNEL_GENERAL, "一般", NotificationManager.IMPORTANCE_DEFAULT),
            NotificationChannel(CHANNEL_REMINDER, "リマインダー", NotificationManager.IMPORTANCE_HIGH).apply {
                enableVibration(true)
                enableLights(true)
            }
        )
        val manager = context.getSystemService(NotificationManager::class.java)
        channels.forEach { manager.createNotificationChannel(it) }
    }

    fun showNotification(title: String, body: String, channelId: String = CHANNEL_GENERAL) {
        val intent = Intent(context, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(context, 0, intent, PendingIntent.FLAG_IMMUTABLE)

        val notification = NotificationCompat.Builder(context, channelId)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()

        if (ActivityCompat.checkSelfPermission(context, Manifest.permission.POST_NOTIFICATIONS)
            == PackageManager.PERMISSION_GRANTED
        ) {
            NotificationManagerCompat.from(context).notify(System.currentTimeMillis().toInt(), notification)
        }
    }
}
```

---

## アクション付き通知

```kotlin
fun showActionNotification(title: String, body: String) {
    val completeIntent = PendingIntent.getBroadcast(
        context, 0,
        Intent(context, NotificationActionReceiver::class.java).apply {
            action = "ACTION_COMPLETE"
        },
        PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(context, CHANNEL_GENERAL)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(body)
        .addAction(R.drawable.ic_check, "完了", completeIntent)
        .addAction(R.drawable.ic_snooze, "後で", snoozeIntent)
        .setAutoCancel(true)
        .build()

    NotificationManagerCompat.from(context).notify(1, notification)
}
```

---

## 通知パーミッション（API 33+）

```kotlin
@Composable
fun NotificationPermissionRequest(onGranted: () -> Unit) {
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) onGranted()
    }

    LaunchedEffect(Unit) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
        } else {
            onGranted()
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| チャネル | `NotificationChannel` |
| 通知表示 | `NotificationCompat.Builder` |
| アクション | `addAction` + PendingIntent |
| パーミッション | `POST_NOTIFICATIONS` (API 33+) |

- `NotificationChannel`でカテゴリ別チャネル作成
- `NotificationCompat`で後方互換通知
- アクションボタンでユーザー操作を追加
- API 33+では`POST_NOTIFICATIONS`パーミッション必須

---

8種類のAndroidアプリテンプレート（通知機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AlarmManager](https://zenn.dev/myougatheaxo/articles/android-compose-alarm-scheduler-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [Firebase Cloud Messaging](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-cloud-messaging-2026)
