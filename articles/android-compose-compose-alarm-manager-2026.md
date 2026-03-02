---
title: "Compose AlarmManager完全ガイド — アラーム設定/リマインダー/正確なタイミング"
emoji: "⏰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "alarm"]
published: true
---

## この記事で学べること

**Compose AlarmManager**（アラーム設定、正確/非正確アラーム、リマインダー、BroadcastReceiver連携）を解説します。

---

## 基本アラーム

```kotlin
class AlarmReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val title = intent.getStringExtra("title") ?: "リマインダー"
        showNotification(context, title)
    }

    private fun showNotification(context: Context, title: String) {
        val notification = NotificationCompat.Builder(context, "reminder")
            .setSmallIcon(R.drawable.ic_alarm)
            .setContentTitle(title)
            .setContentText("予定の時間です")
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()
        NotificationManagerCompat.from(context).notify(System.currentTimeMillis().toInt(), notification)
    }
}

fun setAlarm(context: Context, triggerTimeMillis: Long, title: String, requestCode: Int) {
    val intent = Intent(context, AlarmReceiver::class.java).putExtra("title", title)
    val pendingIntent = PendingIntent.getBroadcast(context, requestCode, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)

    val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && !alarmManager.canScheduleExactAlarms()) {
        alarmManager.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerTimeMillis, pendingIntent)
    } else {
        alarmManager.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerTimeMillis, pendingIntent)
    }
}
```

---

## Compose設定UI

```kotlin
@Composable
fun AlarmSettingScreen() {
    val context = LocalContext.current
    var selectedHour by remember { mutableIntStateOf(8) }
    var selectedMinute by remember { mutableIntStateOf(0) }
    var title by remember { mutableStateOf("朝のリマインダー") }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(value = title, onValueChange = { title = it },
            label = { Text("タイトル") }, modifier = Modifier.fillMaxWidth())

        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("時刻: %02d:%02d".format(selectedHour, selectedMinute),
                style = MaterialTheme.typography.headlineMedium)
        }

        Button(onClick = {
            val calendar = Calendar.getInstance().apply {
                set(Calendar.HOUR_OF_DAY, selectedHour)
                set(Calendar.MINUTE, selectedMinute)
                set(Calendar.SECOND, 0)
                if (before(Calendar.getInstance())) add(Calendar.DAY_OF_MONTH, 1)
            }
            setAlarm(context, calendar.timeInMillis, title, 1001)
        }, Modifier.fillMaxWidth()) {
            Icon(Icons.Default.Alarm, null); Spacer(Modifier.width(8.dp)); Text("アラーム設定")
        }
    }
}
```

---

## キャンセル

```kotlin
fun cancelAlarm(context: Context, requestCode: Int) {
    val intent = Intent(context, AlarmReceiver::class.java)
    val pendingIntent = PendingIntent.getBroadcast(context, requestCode, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
    val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
    alarmManager.cancel(pendingIntent)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `setExactAndAllowWhileIdle` | 正確なアラーム |
| `setAndAllowWhileIdle` | 非正確アラーム |
| `canScheduleExactAlarms` | 権限チェック |
| `cancel` | アラームキャンセル |

- Android 12+は`SCHEDULE_EXACT_ALARM`パーミッション必要
- `setExactAndAllowWhileIdle`でDozeモード中も動作
- `PendingIntent`の`requestCode`でアラームを識別
- 繰り返しは`setRepeating`より`WorkManager`推奨

---

8種類のAndroidアプリテンプレート（リマインダー対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Notification](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
- [WorkManager Periodic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-periodic-2026)
- [Compose BroadcastReceiver](https://zenn.dev/myougatheaxo/articles/android-compose-compose-broadcast-receiver-2026)
