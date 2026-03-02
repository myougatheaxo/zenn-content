---
title: "ローカル通知スケジューリング — WorkManager+通知チャンネル"
emoji: "🔔"
type: "tech"
topics: ["android", "kotlin", "notification", "workmanager"]
published: true
---

## この記事で学べること

**ローカル通知のスケジューリング**（WorkManager定期通知、通知チャンネル管理、アクションボタン）を解説します。

---

## 通知チャンネル管理

```kotlin
object NotificationChannels {

    const val REMINDER = "reminder_channel"
    const val DAILY = "daily_channel"
    const val URGENT = "urgent_channel"

    fun createAll(context: Context) {
        val manager = context.getSystemService<NotificationManager>() ?: return

        val channels = listOf(
            NotificationChannel(REMINDER, "リマインダー", NotificationManager.IMPORTANCE_DEFAULT).apply {
                description = "タスクリマインダー通知"
            },
            NotificationChannel(DAILY, "デイリー通知", NotificationManager.IMPORTANCE_LOW).apply {
                description = "毎日のサマリー通知"
            },
            NotificationChannel(URGENT, "緊急通知", NotificationManager.IMPORTANCE_HIGH).apply {
                description = "重要なお知らせ"
                enableVibration(true)
                enableLights(true)
            }
        )

        manager.createNotificationChannels(channels)
    }
}

// Application
@HiltAndroidApp
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        NotificationChannels.createAll(this)
    }
}
```

---

## WorkManagerで定期通知

```kotlin
class DailyReminderWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val title = inputData.getString("title") ?: "リマインダー"
        val message = inputData.getString("message") ?: "今日のタスクを確認しましょう"

        showNotification(title, message)
        return Result.success()
    }

    private fun showNotification(title: String, message: String) {
        val intent = Intent(applicationContext, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            applicationContext, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(applicationContext, NotificationChannels.REMINDER)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(message)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()

        val manager = applicationContext.getSystemService<NotificationManager>()
        manager?.notify(System.currentTimeMillis().toInt(), notification)
    }
}

// スケジュール設定
fun scheduleDailyReminder(context: Context, hour: Int, minute: Int) {
    val now = Calendar.getInstance()
    val target = Calendar.getInstance().apply {
        set(Calendar.HOUR_OF_DAY, hour)
        set(Calendar.MINUTE, minute)
        set(Calendar.SECOND, 0)
        if (before(now)) add(Calendar.DAY_OF_YEAR, 1)
    }

    val delay = target.timeInMillis - now.timeInMillis

    val request = OneTimeWorkRequestBuilder<DailyReminderWorker>()
        .setInitialDelay(delay, TimeUnit.MILLISECONDS)
        .setInputData(workDataOf(
            "title" to "おはようございます",
            "message" to "今日のタスクを確認しましょう"
        ))
        .addTag("daily_reminder")
        .build()

    WorkManager.getInstance(context)
        .enqueueUniqueWork("daily_reminder", ExistingWorkPolicy.REPLACE, request)
}
```

---

## アクション付き通知

```kotlin
fun showActionNotification(context: Context, taskId: String, taskTitle: String) {
    // 完了アクション
    val completeIntent = Intent(context, NotificationActionReceiver::class.java).apply {
        action = "ACTION_COMPLETE"
        putExtra("task_id", taskId)
    }
    val completePending = PendingIntent.getBroadcast(
        context, taskId.hashCode(), completeIntent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    // スヌーズアクション
    val snoozeIntent = Intent(context, NotificationActionReceiver::class.java).apply {
        action = "ACTION_SNOOZE"
        putExtra("task_id", taskId)
    }
    val snoozePending = PendingIntent.getBroadcast(
        context, taskId.hashCode() + 1, snoozeIntent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(context, NotificationChannels.REMINDER)
        .setSmallIcon(R.drawable.ic_task)
        .setContentTitle("タスクリマインダー")
        .setContentText(taskTitle)
        .addAction(R.drawable.ic_check, "完了", completePending)
        .addAction(R.drawable.ic_snooze, "10分後", snoozePending)
        .setAutoCancel(true)
        .build()

    context.getSystemService<NotificationManager>()?.notify(taskId.hashCode(), notification)
}

class NotificationActionReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val taskId = intent.getStringExtra("task_id") ?: return
        when (intent.action) {
            "ACTION_COMPLETE" -> {
                // タスク完了処理
                context.getSystemService<NotificationManager>()?.cancel(taskId.hashCode())
            }
            "ACTION_SNOOZE" -> {
                context.getSystemService<NotificationManager>()?.cancel(taskId.hashCode())
                // 10分後に再通知
            }
        }
    }
}
```

---

## Compose UI でスケジュール管理

```kotlin
@Composable
fun ReminderSettingsScreen() {
    val context = LocalContext.current
    var hour by remember { mutableIntStateOf(8) }
    var minute by remember { mutableIntStateOf(0) }
    var enabled by remember { mutableStateOf(true) }

    Column(Modifier.padding(16.dp)) {
        Text("リマインダー設定", style = MaterialTheme.typography.headlineMedium)

        Spacer(Modifier.height(16.dp))

        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("毎日のリマインダー", Modifier.weight(1f))
            Switch(checked = enabled, onCheckedChange = { isOn ->
                enabled = isOn
                if (isOn) {
                    scheduleDailyReminder(context, hour, minute)
                } else {
                    WorkManager.getInstance(context).cancelUniqueWork("daily_reminder")
                }
            })
        }

        if (enabled) {
            Spacer(Modifier.height(8.dp))
            Text(
                text = String.format("%02d:%02d", hour, minute),
                style = MaterialTheme.typography.displayMedium
            )
        }
    }
}
```

---

## まとめ

- `NotificationChannel`をアプリ起動時に一括作成
- `WorkManager` + `OneTimeWorkRequest`でスケジュール通知
- `enqueueUniqueWork`で重複防止
- `addAction`で通知にアクションボタン追加
- `BroadcastReceiver`でアクション処理
- Android 13+は`POST_NOTIFICATIONS`パーミッション必須

---

8種類のAndroidアプリテンプレート（通知機能実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [通知チャンネル](https://zenn.dev/myougatheaxo/articles/android-compose-notification-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-2026)
- [AlarmManager](https://zenn.dev/myougatheaxo/articles/android-compose-alarm-notification-2026)
