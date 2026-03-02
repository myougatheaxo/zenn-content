---
title: "Android通知(Notification)完全ガイド — チャンネル/スタイル/アクション"
emoji: "🔔"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "notification"]
published: true
---

## この記事で学べること

Androidの**通知システム**（チャンネル、スタイル、アクション、進捗表示）を解説します。

---

## 通知チャンネル作成

```kotlin
class NotificationHelper(private val context: Context) {
    companion object {
        const val CHANNEL_GENERAL = "general"
        const val CHANNEL_IMPORTANT = "important"
    }

    fun createChannels() {
        val general = NotificationChannel(
            CHANNEL_GENERAL,
            "一般通知",
            NotificationManager.IMPORTANCE_DEFAULT
        ).apply {
            description = "一般的なお知らせ"
        }

        val important = NotificationChannel(
            CHANNEL_IMPORTANT,
            "重要な通知",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description = "重要なアラート"
            enableVibration(true)
            enableLights(true)
        }

        val manager = context.getSystemService(NotificationManager::class.java)
        manager.createNotificationChannels(listOf(general, important))
    }
}
```

---

## 基本通知

```kotlin
fun showSimpleNotification(context: Context, title: String, body: String) {
    val intent = Intent(context, MainActivity::class.java).apply {
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
    }
    val pendingIntent = PendingIntent.getActivity(
        context, 0, intent, PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(context, CHANNEL_GENERAL)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(body)
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .build()

    NotificationManagerCompat.from(context).notify(1, notification)
}
```

---

## BigTextStyle / BigPictureStyle

```kotlin
// 長文通知
fun showBigTextNotification(context: Context) {
    val notification = NotificationCompat.Builder(context, CHANNEL_GENERAL)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("更新情報")
        .setContentText("アプリが更新されました")
        .setStyle(
            NotificationCompat.BigTextStyle()
                .bigText("バージョン2.0では以下の機能が追加されました：\n・ダークモード対応\n・パフォーマンス改善\n・新しいウィジェット")
        )
        .build()

    NotificationManagerCompat.from(context).notify(2, notification)
}

// 画像付き通知
fun showBigPictureNotification(context: Context, bitmap: Bitmap) {
    val notification = NotificationCompat.Builder(context, CHANNEL_GENERAL)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("写真が共有されました")
        .setStyle(
            NotificationCompat.BigPictureStyle()
                .bigPicture(bitmap)
                .bigLargeIcon(null as Bitmap?)
        )
        .build()

    NotificationManagerCompat.from(context).notify(3, notification)
}
```

---

## アクションボタン

```kotlin
fun showActionNotification(context: Context) {
    val approveIntent = PendingIntent.getBroadcast(
        context, 0,
        Intent(context, NotificationReceiver::class.java).apply {
            action = "APPROVE"
        },
        PendingIntent.FLAG_IMMUTABLE
    )

    val rejectIntent = PendingIntent.getBroadcast(
        context, 1,
        Intent(context, NotificationReceiver::class.java).apply {
            action = "REJECT"
        },
        PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(context, CHANNEL_IMPORTANT)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("承認リクエスト")
        .setContentText("新しいリクエストがあります")
        .addAction(R.drawable.ic_check, "承認", approveIntent)
        .addAction(R.drawable.ic_close, "却下", rejectIntent)
        .build()

    NotificationManagerCompat.from(context).notify(4, notification)
}
```

---

## まとめ

- `NotificationChannel`でAndroid 8+のチャンネル作成
- `NotificationCompat.Builder`で後方互換性のある通知
- `BigTextStyle`/`BigPictureStyle`で拡張表示
- `addAction`でアクションボタン追加
- `PendingIntent.FLAG_IMMUTABLE`でセキュリティ対応
- Android 13+は`POST_NOTIFICATIONS`パーミッション必須

---

8種類のAndroidアプリテンプレート（通知機能実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager連携](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-2026)
- [パーミッション処理](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
- [AlarmManager活用](https://zenn.dev/myougatheaxo/articles/android-compose-countdown-alarm-2026)
