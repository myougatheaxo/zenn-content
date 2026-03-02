---
title: "AlarmManager + 通知ガイド — 正確なスケジューリング"
emoji: "⏰"
type: "tech"
topics: ["android", "kotlin", "alarm", "notification"]
published: true
---

## この記事で学べること

**AlarmManager**と通知の連携（正確なアラーム、BroadcastReceiver、再起動後の復元）を解説します。

---

## AlarmManager 基本

```kotlin
class AlarmScheduler(private val context: Context) {

    private val alarmManager = context.getSystemService<AlarmManager>()!!

    fun scheduleExact(timeInMillis: Long, alarmId: Int, title: String) {
        val intent = Intent(context, AlarmReceiver::class.java).apply {
            putExtra("alarm_id", alarmId)
            putExtra("title", title)
        }

        val pendingIntent = PendingIntent.getBroadcast(
            context,
            alarmId,
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        // Android 12+ で正確なアラームには権限が必要
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            if (alarmManager.canScheduleExactAlarms()) {
                alarmManager.setExactAndAllowWhileIdle(
                    AlarmManager.RTC_WAKEUP, timeInMillis, pendingIntent
                )
            }
        } else {
            alarmManager.setExactAndAllowWhileIdle(
                AlarmManager.RTC_WAKEUP, timeInMillis, pendingIntent
            )
        }
    }

    fun cancel(alarmId: Int) {
        val intent = Intent(context, AlarmReceiver::class.java)
        val pendingIntent = PendingIntent.getBroadcast(
            context, alarmId, intent,
            PendingIntent.FLAG_NO_CREATE or PendingIntent.FLAG_IMMUTABLE
        )
        pendingIntent?.let { alarmManager.cancel(it) }
    }
}
```

---

## BroadcastReceiver

```kotlin
class AlarmReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        val alarmId = intent.getIntExtra("alarm_id", 0)
        val title = intent.getStringExtra("title") ?: "アラーム"

        showNotification(context, alarmId, title)
    }

    private fun showNotification(context: Context, id: Int, title: String) {
        val channelId = "alarm_channel"

        // チャンネル作成（Android 8+）
        val channel = NotificationChannel(
            channelId, "アラーム",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            enableVibration(true)
            vibrationPattern = longArrayOf(0, 500, 200, 500)
        }
        val notificationManager = context.getSystemService<NotificationManager>()!!
        notificationManager.createNotificationChannel(channel)

        // 通知作成
        val notification = NotificationCompat.Builder(context, channelId)
            .setSmallIcon(R.drawable.ic_alarm)
            .setContentTitle(title)
            .setContentText("アラームの時刻です")
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .setCategory(NotificationCompat.CATEGORY_ALARM)
            .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)
            .build()

        notificationManager.notify(id, notification)
    }
}
```

---

## 繰り返しアラーム

```kotlin
fun scheduleRepeating(
    context: Context,
    intervalMillis: Long,
    startTimeMillis: Long,
    alarmId: Int
) {
    val alarmManager = context.getSystemService<AlarmManager>()!!
    val intent = Intent(context, AlarmReceiver::class.java).apply {
        putExtra("alarm_id", alarmId)
    }
    val pendingIntent = PendingIntent.getBroadcast(
        context, alarmId, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    // 非正確な繰り返し（バッテリー効率良い）
    alarmManager.setRepeating(
        AlarmManager.RTC_WAKEUP,
        startTimeMillis,
        intervalMillis,
        pendingIntent
    )
}

// 毎日同じ時刻
fun scheduleDailyAlarm(context: Context, hour: Int, minute: Int, alarmId: Int) {
    val calendar = Calendar.getInstance().apply {
        set(Calendar.HOUR_OF_DAY, hour)
        set(Calendar.MINUTE, minute)
        set(Calendar.SECOND, 0)
        if (timeInMillis <= System.currentTimeMillis()) {
            add(Calendar.DAY_OF_YEAR, 1) // 過去なら翌日
        }
    }

    scheduleRepeating(
        context,
        AlarmManager.INTERVAL_DAY,
        calendar.timeInMillis,
        alarmId
    )
}
```

---

## 再起動後の復元

```kotlin
// AndroidManifest.xml
// <receiver android:name=".BootReceiver"
//     android:exported="true">
//     <intent-filter>
//         <action android:name="android.intent.action.BOOT_COMPLETED" />
//     </intent-filter>
// </receiver>

class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // DataStore/Roomからアラーム情報を読み取って再設定
            val scope = CoroutineScope(Dispatchers.IO)
            scope.launch {
                val alarms = AlarmRepository(context).getActiveAlarms()
                val scheduler = AlarmScheduler(context)
                alarms.forEach { alarm ->
                    if (alarm.timeInMillis > System.currentTimeMillis()) {
                        scheduler.scheduleExact(alarm.timeInMillis, alarm.id, alarm.title)
                    }
                }
            }
        }
    }
}
```

---

## Compose UI連携

```kotlin
@Composable
fun AlarmSettingScreen() {
    val context = LocalContext.current
    var selectedTime by remember { mutableStateOf(LocalTime.now()) }

    Column(Modifier.padding(16.dp)) {
        Text("アラーム設定", style = MaterialTheme.typography.headlineMedium)

        Spacer(Modifier.height(16.dp))

        // 時刻表示
        Text(
            text = selectedTime.format(DateTimeFormatter.ofPattern("HH:mm")),
            style = MaterialTheme.typography.displayLarge
        )

        Spacer(Modifier.height(24.dp))

        Button(onClick = {
            val calendar = Calendar.getInstance().apply {
                set(Calendar.HOUR_OF_DAY, selectedTime.hour)
                set(Calendar.MINUTE, selectedTime.minute)
                set(Calendar.SECOND, 0)
                if (timeInMillis <= System.currentTimeMillis()) {
                    add(Calendar.DAY_OF_YEAR, 1)
                }
            }

            AlarmScheduler(context).scheduleExact(
                timeInMillis = calendar.timeInMillis,
                alarmId = 1,
                title = "起床アラーム"
            )
        }) {
            Icon(Icons.Default.Alarm, null)
            Spacer(Modifier.width(8.dp))
            Text("アラームを設定")
        }
    }
}
```

---

## まとめ

- `setExactAndAllowWhileIdle`で正確なアラーム（Dozeモード対応）
- Android 12+は`SCHEDULE_EXACT_ALARM`権限必要
- `BroadcastReceiver`でアラーム受信→通知表示
- `BOOT_COMPLETED`で再起動後にアラーム再設定
- `PendingIntent.FLAG_IMMUTABLE`はAndroid 12+必須
- 繰り返しには`setRepeating`（非正確）or 毎回次回を設定

---

8種類のAndroidアプリテンプレート（アラーム機能実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [通知チャンネル](https://zenn.dev/myougatheaxo/articles/android-compose-notification-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-2026)
- [Foreground Service](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
