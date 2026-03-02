---
title: "通知の応用パターン — BigPicture/進捗/グループ/返信アクション"
emoji: "🔔"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "notification"]
published: true
---

## この記事で学べること

**通知の応用**（BigPictureStyle、進捗バー、グループ通知、ダイレクト返信、バブル、メディアスタイル）を解説します。

---

## BigPictureStyle

```kotlin
fun showBigPictureNotification(context: Context, bitmap: Bitmap) {
    val notification = NotificationCompat.Builder(context, "media")
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("新しい写真")
        .setContentText("共有された写真があります")
        .setStyle(
            NotificationCompat.BigPictureStyle()
                .bigPicture(bitmap)
                .bigLargeIcon(null as Bitmap?)
        )
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .build()

    NotificationManagerCompat.from(context).notify(1, notification)
}
```

---

## 進捗バー通知

```kotlin
fun showProgressNotification(context: Context, progress: Int, max: Int) {
    val builder = NotificationCompat.Builder(context, "download")
        .setSmallIcon(R.drawable.ic_download)
        .setContentTitle("ダウンロード中")
        .setOngoing(true)

    if (progress < max) {
        builder.setProgress(max, progress, false)
            .setContentText("$progress / $max")
    } else {
        builder.setProgress(0, 0, false)
            .setContentText("完了")
            .setOngoing(false)
    }

    NotificationManagerCompat.from(context).notify(2, builder.build())
}
```

---

## グループ通知

```kotlin
fun showGroupNotification(context: Context, messages: List<Message>) {
    val groupKey = "com.example.MESSAGES"

    messages.forEachIndexed { index, message ->
        val notification = NotificationCompat.Builder(context, "messages")
            .setSmallIcon(R.drawable.ic_message)
            .setContentTitle(message.sender)
            .setContentText(message.text)
            .setGroup(groupKey)
            .build()
        NotificationManagerCompat.from(context).notify(100 + index, notification)
    }

    val summary = NotificationCompat.Builder(context, "messages")
        .setSmallIcon(R.drawable.ic_message)
        .setContentTitle("${messages.size}件の新しいメッセージ")
        .setStyle(
            NotificationCompat.InboxStyle()
                .also { style ->
                    messages.forEach { style.addLine("${it.sender}: ${it.text}") }
                }
                .setSummaryText("${messages.size}件の新しいメッセージ")
        )
        .setGroup(groupKey)
        .setGroupSummary(true)
        .build()
    NotificationManagerCompat.from(context).notify(99, summary)
}
```

---

## ダイレクト返信

```kotlin
fun showReplyNotification(context: Context, conversationId: String) {
    val remoteInput = RemoteInput.Builder("key_reply")
        .setLabel("返信を入力")
        .build()

    val replyPendingIntent = PendingIntent.getBroadcast(
        context, 0,
        Intent(context, ReplyReceiver::class.java).putExtra("conversation_id", conversationId),
        PendingIntent.FLAG_MUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
    )

    val replyAction = NotificationCompat.Action.Builder(
        R.drawable.ic_reply, "返信", replyPendingIntent
    ).addRemoteInput(remoteInput).build()

    val notification = NotificationCompat.Builder(context, "messages")
        .setSmallIcon(R.drawable.ic_message)
        .setContentTitle("新しいメッセージ")
        .setContentText("タップして返信")
        .addAction(replyAction)
        .build()

    NotificationManagerCompat.from(context).notify(3, notification)
}

class ReplyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val reply = RemoteInput.getResultsFromIntent(intent)?.getCharSequence("key_reply")
        // 返信処理
    }
}
```

---

## まとめ

| スタイル | 用途 |
|---------|------|
| BigPicture | 画像付き通知 |
| Progress | ダウンロード進捗 |
| InboxStyle | グループ通知 |
| RemoteInput | ダイレクト返信 |
| MessagingStyle | チャット形式 |

- `BigPictureStyle`で画像付き展開通知
- `setProgress()`で進捗バー表示
- `setGroup()`で関連通知をグループ化
- `RemoteInput`で通知から直接返信

---

8種類のAndroidアプリテンプレート（通知機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [通知基本](https://zenn.dev/myougatheaxo/articles/android-compose-notification-2026)
- [FCM](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-messaging-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-chain-2026)
