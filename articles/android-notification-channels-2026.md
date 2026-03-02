---
title: "通知チャネル実装ガイド — ローカル通知の送信とカスタマイズ"
emoji: "🔔"
type: "tech"
topics: ["android", "kotlin", "notification", "channels"]
published: true
---

## この記事で学べること

Androidの**ローカル通知**（NotificationChannel・通知スタイル・アクション）の実装方法を解説します。

---

## 通知チャネルの作成

```kotlin
class NotificationHelper(private val context: Context) {

    companion object {
        const val CHANNEL_GENERAL = "general"
        const val CHANNEL_IMPORTANT = "important"
        const val CHANNEL_SILENT = "silent"
    }

    fun createChannels() {
        val channels = listOf(
            NotificationChannel(
                CHANNEL_GENERAL, "一般通知",
                NotificationManager.IMPORTANCE_DEFAULT
            ).apply {
                description = "アプリからのお知らせ"
            },
            NotificationChannel(
                CHANNEL_IMPORTANT, "重要な通知",
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "重要なアラート"
                enableVibration(true)
                enableLights(true)
            },
            NotificationChannel(
                CHANNEL_SILENT, "サイレント通知",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "バックグラウンド処理の通知"
                setSound(null, null)
            }
        )

        val manager = context.getSystemService(NotificationManager::class.java)
        channels.forEach { manager.createNotificationChannel(it) }
    }
}
```

---

## 基本の通知送信

```kotlin
fun sendNotification(context: Context, title: String, message: String) {
    val intent = Intent(context, MainActivity::class.java).apply {
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
    }
    val pendingIntent = PendingIntent.getActivity(
        context, 0, intent, PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(context, NotificationHelper.CHANNEL_GENERAL)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(message)
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .build()

    if (ActivityCompat.checkSelfPermission(
            context, Manifest.permission.POST_NOTIFICATIONS
        ) == PackageManager.PERMISSION_GRANTED
    ) {
        NotificationManagerCompat.from(context).notify(1, notification)
    }
}
```

---

## アクション付き通知

```kotlin
fun sendActionNotification(context: Context) {
    val replyIntent = PendingIntent.getBroadcast(
        context, 0,
        Intent(context, NotificationReceiver::class.java).apply {
            action = "REPLY"
        },
        PendingIntent.FLAG_IMMUTABLE
    )

    val dismissIntent = PendingIntent.getBroadcast(
        context, 1,
        Intent(context, NotificationReceiver::class.java).apply {
            action = "DISMISS"
        },
        PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(context, NotificationHelper.CHANNEL_IMPORTANT)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("新しいメッセージ")
        .setContentText("田中さんからメッセージが届きました")
        .addAction(R.drawable.ic_reply, "返信", replyIntent)
        .addAction(R.drawable.ic_close, "無視", dismissIntent)
        .setAutoCancel(true)
        .build()

    NotificationManagerCompat.from(context).notify(2, notification)
}
```

---

## 進捗通知

```kotlin
fun sendProgressNotification(context: Context, progress: Int) {
    val notification = NotificationCompat.Builder(context, NotificationHelper.CHANNEL_SILENT)
        .setSmallIcon(R.drawable.ic_download)
        .setContentTitle("ダウンロード中")
        .setContentText("$progress%")
        .setProgress(100, progress, false)
        .setOngoing(true)
        .build()

    NotificationManagerCompat.from(context).notify(3, notification)

    // 完了時
    if (progress == 100) {
        val complete = NotificationCompat.Builder(context, NotificationHelper.CHANNEL_GENERAL)
            .setSmallIcon(R.drawable.ic_done)
            .setContentTitle("ダウンロード完了")
            .setContentText("ファイルの準備ができました")
            .setAutoCancel(true)
            .build()
        NotificationManagerCompat.from(context).notify(3, complete)
    }
}
```

---

## Composeからの通知送信

```kotlin
@Composable
fun NotificationScreen() {
    val context = LocalContext.current

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            sendNotification(context, "テスト", "通知の送信テストです")
        }
    }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Button(onClick = {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                permissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
            } else {
                sendNotification(context, "テスト", "通知テスト")
            }
        }) {
            Text("通知を送信")
        }
    }
}
```

---

## まとめ

- `NotificationChannel`で通知の重要度/サウンド/バイブ設定
- `NotificationCompat.Builder`で通知作成
- `PendingIntent`でタップ時のアクション
- `addAction()`でボタン付き通知
- `setProgress()`で進捗バー表示
- Android 13+は`POST_NOTIFICATIONS`パーミッション必須

---

8種類のAndroidアプリテンプレート（通知機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [FCMプッシュ通知ガイド](https://zenn.dev/myougatheaxo/articles/android-fcm-push-notification-2026)
- [AlarmManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-alarm-scheduler-2026)
- [Foreground Serviceガイド](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
