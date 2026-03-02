---
title: "AlarmManager & スケジュール実行ガイド — 正確なタイミングで処理を実行"
emoji: "⏰"
type: "tech"
topics: ["android", "kotlin", "alarm", "scheduler"]
published: true
---

## この記事で学べること

AlarmManagerで**正確な時刻に通知やタスクを実行**する方法を解説します。WorkManagerとの使い分けも明確にします。

---

## AlarmManager vs WorkManager

| 特徴 | AlarmManager | WorkManager |
|------|-------------|-------------|
| 精度 | 正確な時刻 | おおよそ |
| 用途 | アラーム、リマインダー | バックグラウンド処理 |
| Dozeモード | `setExactAndAllowWhileIdle` | 制約設定 |
| 繰り返し | `setRepeating` | `PeriodicWorkRequest` |

**原則**: 正確な時刻が必要 → AlarmManager、それ以外 → WorkManager

---

## 基本のアラーム設定

```kotlin
class AlarmScheduler(private val context: Context) {

    private val alarmManager = context.getSystemService<AlarmManager>()

    fun scheduleAlarm(timeInMillis: Long, requestCode: Int) {
        val intent = Intent(context, AlarmReceiver::class.java).apply {
            putExtra("request_code", requestCode)
        }

        val pendingIntent = PendingIntent.getBroadcast(
            context,
            requestCode,
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        alarmManager?.setExactAndAllowWhileIdle(
            AlarmManager.RTC_WAKEUP,
            timeInMillis,
            pendingIntent
        )
    }

    fun cancelAlarm(requestCode: Int) {
        val intent = Intent(context, AlarmReceiver::class.java)
        val pendingIntent = PendingIntent.getBroadcast(
            context,
            requestCode,
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        alarmManager?.cancel(pendingIntent)
    }
}
```

---

## BroadcastReceiver

```kotlin
class AlarmReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val requestCode = intent.getIntExtra("request_code", 0)

        // 通知を表示
        val notificationManager =
            context.getSystemService<NotificationManager>()

        val notification = NotificationCompat.Builder(context, "alarm_channel")
            .setSmallIcon(R.drawable.ic_alarm)
            .setContentTitle("リマインダー")
            .setContentText("設定した時刻になりました")
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()

        notificationManager?.notify(requestCode, notification)
    }
}
```

---

## AndroidManifest.xml

```xml
<receiver
    android:name=".AlarmReceiver"
    android:exported="false" />

<!-- Android 12+: 正確なアラームの権限 -->
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />

<!-- Android 14+: USE_EXACT_ALARM（アラームアプリ向け） -->
<uses-permission android:name="android.permission.USE_EXACT_ALARM" />
```

---

## 毎日のリマインダー

```kotlin
fun scheduleDailyReminder(hour: Int, minute: Int) {
    val calendar = Calendar.getInstance().apply {
        set(Calendar.HOUR_OF_DAY, hour)
        set(Calendar.MINUTE, minute)
        set(Calendar.SECOND, 0)

        // 過去の時刻なら翌日にセット
        if (timeInMillis <= System.currentTimeMillis()) {
            add(Calendar.DAY_OF_YEAR, 1)
        }
    }

    val intent = Intent(context, AlarmReceiver::class.java)
    val pendingIntent = PendingIntent.getBroadcast(
        context, DAILY_REMINDER_CODE, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    alarmManager?.setRepeating(
        AlarmManager.RTC_WAKEUP,
        calendar.timeInMillis,
        AlarmManager.INTERVAL_DAY,
        pendingIntent
    )
}
```

---

## 再起動後のアラーム復元

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // DataStoreからアラーム設定を読み込んで再セット
            val scheduler = AlarmScheduler(context)
            // scheduler.restoreAlarms()
        }
    }
}
```

```xml
<receiver
    android:name=".BootReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>

<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

---

## Composeでの時刻選択

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimePickerDialog(
    onTimeSelected: (Int, Int) -> Unit,
    onDismiss: () -> Unit
) {
    val timePickerState = rememberTimePickerState()

    AlertDialog(
        onDismissRequest = onDismiss,
        confirmButton = {
            TextButton(onClick = {
                onTimeSelected(timePickerState.hour, timePickerState.minute)
            }) { Text("設定") }
        },
        text = { TimePicker(state = timePickerState) }
    )
}
```

---

## まとめ

- 正確な時刻 → `AlarmManager.setExactAndAllowWhileIdle`
- バックグラウンド処理 → WorkManager
- `PendingIntent.FLAG_IMMUTABLE`は必須（Android 12+）
- 再起動後は`BOOT_COMPLETED`でアラーム復元
- Android 12+では`SCHEDULE_EXACT_ALARM`権限が必要

---

8種類のAndroidアプリテンプレート（スケジュール機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Android通知の実装ガイド](https://zenn.dev/myougatheaxo/articles/android-notifications-guide-2026)
- [WorkManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-workmanager-guide-2026)
- [パーミッション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-permission-runtime-2026)
