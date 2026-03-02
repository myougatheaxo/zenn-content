---
title: "通知完全ガイド — NotificationCompat/チャンネル/カスタム通知/Compose連携"
emoji: "🔔"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "notification"]
published: true
---

## この記事で学べること

**通知**（NotificationCompat、チャンネル作成、カスタム通知、権限リクエスト）を解説します。

---

## 通知チャンネル

```kotlin
object NotificationHelper {
    const val CHANNEL_DEFAULT = "default"
    const val CHANNEL_IMPORTANT = "important"

    fun createChannels(context: Context) {
        val defaultChannel = NotificationChannelCompat.Builder(
            CHANNEL_DEFAULT, NotificationManagerCompat.IMPORTANCE_DEFAULT
        ).setName("一般通知").setDescription("一般的な通知").build()

        val importantChannel = NotificationChannelCompat.Builder(
            CHANNEL_IMPORTANT, NotificationManagerCompat.IMPORTANCE_HIGH
        ).setName("重要な通知").setDescription("見逃せない通知").build()

        NotificationManagerCompat.from(context)
            .createNotificationChannelsCompat(listOf(defaultChannel, importantChannel))
    }

    fun showNotification(context: Context, title: String, message: String) {
        val notification = NotificationCompat.Builder(context, CHANNEL_DEFAULT)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(message)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .setAutoCancel(true)
            .build()

        if (ActivityCompat.checkSelfPermission(context, POST_NOTIFICATIONS) == PERMISSION_GRANTED) {
            NotificationManagerCompat.from(context).notify(System.currentTimeMillis().toInt(), notification)
        }
    }
}
```

---

## Compose連携

```kotlin
@Composable
fun NotificationScreen() {
    val context = LocalContext.current
    val permissionState = rememberPermissionState(Manifest.permission.POST_NOTIFICATIONS)

    LaunchedEffect(Unit) { NotificationHelper.createChannels(context) }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        if (!permissionState.status.isGranted) {
            Card(colors = CardDefaults.cardColors(containerColor = Color(0xFFFFF3E0))) {
                Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.Warning, null, tint = Color(0xFFFF9800))
                    Spacer(Modifier.width(8.dp))
                    Column(Modifier.weight(1f)) {
                        Text("通知権限が必要です")
                        TextButton(onClick = { permissionState.launchPermissionRequest() }) {
                            Text("許可する")
                        }
                    }
                }
            }
        }

        Button(onClick = {
            NotificationHelper.showNotification(context, "テスト通知", "通知が正常に動作しています")
        }, enabled = permissionState.status.isGranted) {
            Icon(Icons.Default.Notifications, null)
            Spacer(Modifier.width(8.dp))
            Text("通知を送信")
        }
    }
}
```

---

## アクション付き通知

```kotlin
fun showActionNotification(context: Context) {
    val intent = Intent(context, MainActivity::class.java).apply {
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
    }
    val pendingIntent = PendingIntent.getActivity(context, 0, intent, PendingIntent.FLAG_IMMUTABLE)

    val notification = NotificationCompat.Builder(context, NotificationHelper.CHANNEL_DEFAULT)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("新しいメッセージ")
        .setContentText("山田さんからメッセージが届きました")
        .setContentIntent(pendingIntent)
        .addAction(R.drawable.ic_reply, "返信", pendingIntent)
        .addAction(R.drawable.ic_dismiss, "既読", pendingIntent)
        .setStyle(NotificationCompat.BigTextStyle().bigText("こんにちは。明日の会議の件でご連絡しました。"))
        .build()

    if (ActivityCompat.checkSelfPermission(context, POST_NOTIFICATIONS) == PERMISSION_GRANTED) {
        NotificationManagerCompat.from(context).notify(1, notification)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `NotificationCompat` | 通知構築 |
| `NotificationChannel` | チャンネル定義 |
| `addAction` | アクションボタン |
| `POST_NOTIFICATIONS` | Android 13+権限 |

- Android 8.0+はNotificationChannel必須
- Android 13+はPOST_NOTIFICATIONS権限必須
- `NotificationCompat`で後方互換性を確保
- アクションボタンでインタラクティブ通知

---

8種類のAndroidアプリテンプレート（通知対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-chain-2026)
- [パーミッション管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [Accompanist Permissions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-accompanist-permissions-2026)
