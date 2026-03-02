---
title: "Android通知の実装ガイド — チャンネル作成からパーミッション対応まで"
emoji: "🔔"
type: "tech"
topics: ["android", "kotlin", "notification", "jetpackcompose"]
published: true
---

## この記事で学べること

Android 8.0以降は通知チャンネルが必須、Android 13以降は通知パーミッションのリクエストが必要。AIが生成するアプリに通知機能を追加する方法を解説します。

---

## 通知の3ステップ

### Step 1: 通知チャンネルの作成（Android 8.0+）

```kotlin
// Application.kt の onCreate() で
private fun createNotificationChannel() {
    val channel = NotificationChannel(
        "habit_reminder",           // チャンネルID
        "習慣リマインダー",           // ユーザーに表示される名前
        NotificationManager.IMPORTANCE_DEFAULT
    ).apply {
        description = "習慣の達成リマインドを通知します"
    }

    val manager = getSystemService(NotificationManager::class.java)
    manager.createNotificationChannel(channel)
}
```

### Step 2: パーミッションのリクエスト（Android 13+）

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

```kotlin
@Composable
fun NotificationPermissionRequest() {
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            // 通知許可された
        }
    }

    LaunchedEffect(Unit) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
}
```

### Step 3: 通知の送信

```kotlin
fun sendNotification(context: Context, title: String, message: String) {
    val notification = NotificationCompat.Builder(context, "habit_reminder")
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(message)
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setAutoCancel(true)
        .build()

    val manager = NotificationManagerCompat.from(context)
    if (ActivityCompat.checkSelfPermission(
            context,
            Manifest.permission.POST_NOTIFICATIONS
        ) == PackageManager.PERMISSION_GRANTED
    ) {
        manager.notify(1, notification)
    }
}
```

---

## タップで画面を開く

```kotlin
val intent = Intent(context, MainActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
}
val pendingIntent = PendingIntent.getActivity(
    context, 0, intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

val notification = NotificationCompat.Builder(context, "habit_reminder")
    .setSmallIcon(R.drawable.ic_notification)
    .setContentTitle("習慣の時間です")
    .setContentText("今日のランニングを記録しましょう")
    .setContentIntent(pendingIntent)
    .setAutoCancel(true)
    .build()
```

---

## 定期通知（AlarmManager）

```kotlin
fun scheduleDaily(context: Context, hour: Int, minute: Int) {
    val alarmManager = context.getSystemService(AlarmManager::class.java)

    val intent = Intent(context, NotificationReceiver::class.java)
    val pendingIntent = PendingIntent.getBroadcast(
        context, 0, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    val calendar = Calendar.getInstance().apply {
        set(Calendar.HOUR_OF_DAY, hour)
        set(Calendar.MINUTE, minute)
        set(Calendar.SECOND, 0)
        if (before(Calendar.getInstance())) {
            add(Calendar.DAY_OF_YEAR, 1)
        }
    }

    alarmManager.setRepeating(
        AlarmManager.RTC_WAKEUP,
        calendar.timeInMillis,
        AlarmManager.INTERVAL_DAY,
        pendingIntent
    )
}
```

```kotlin
class NotificationReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        sendNotification(context, "リマインダー", "今日の習慣を記録しましょう")
    }
}
```

---

## 通知の重要度

| レベル | 動作 |
|--------|------|
| IMPORTANCE_HIGH | ヘッドアップ通知（画面上部に表示） |
| IMPORTANCE_DEFAULT | 通知バーに表示 + 音 |
| IMPORTANCE_LOW | 通知バーに表示のみ |
| IMPORTANCE_MIN | 折りたたみ表示 |

---

## まとめ

- 通知チャンネルはAndroid 8.0+で必須
- `POST_NOTIFICATIONS`パーミッションはAndroid 13+で必須
- `NotificationCompat.Builder`で通知を構築
- `PendingIntent`でタップ時のアクションを設定
- `AlarmManager`で定期通知をスケジュール

---

8種類のAndroidアプリテンプレート（通知機能の追加も容易なMVVM設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [アクセシビリティ対応入門](https://zenn.dev/myougatheaxo/articles/android-accessibility-basics-2026)
