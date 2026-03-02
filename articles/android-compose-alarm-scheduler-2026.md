---
title: "アラーム/スケジューラー完全ガイド — AlarmManager/WorkManager/正確なタイマー"
emoji: "⏰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "scheduler"]
published: true
---

## この記事で学べること

**アラーム/スケジューラー**（AlarmManager、正確なアラーム、WorkManager定期実行、通知連携）を解説します。

---

## AlarmManager

```kotlin
class AlarmScheduler @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager

    fun scheduleExact(alarmId: Int, timeMillis: Long) {
        val intent = Intent(context, AlarmReceiver::class.java).apply {
            putExtra("alarm_id", alarmId)
        }
        val pendingIntent = PendingIntent.getBroadcast(
            context, alarmId, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            if (alarmManager.canScheduleExactAlarms()) {
                alarmManager.setExactAndAllowWhileIdle(
                    AlarmManager.RTC_WAKEUP, timeMillis, pendingIntent
                )
            }
        } else {
            alarmManager.setExactAndAllowWhileIdle(
                AlarmManager.RTC_WAKEUP, timeMillis, pendingIntent
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

## AlarmReceiver

```kotlin
class AlarmReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val alarmId = intent.getIntExtra("alarm_id", 0)

        // 通知を表示
        val notificationManager = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

        val notification = NotificationCompat.Builder(context, "alarms")
            .setSmallIcon(R.drawable.ic_alarm)
            .setContentTitle("アラーム")
            .setContentText("設定した時刻です")
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setCategory(NotificationCompat.CATEGORY_ALARM)
            .setAutoCancel(true)
            .build()

        notificationManager.notify(alarmId, notification)
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun AlarmSettingScreen(viewModel: AlarmViewModel = hiltViewModel()) {
    val alarms by viewModel.alarms.collectAsStateWithLifecycle()
    var showTimePicker by remember { mutableStateOf(false) }

    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = { showTimePicker = true }) {
                Icon(Icons.Default.Add, "追加")
            }
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(alarms) { alarm ->
                AlarmItem(
                    alarm = alarm,
                    onToggle = { viewModel.toggleAlarm(alarm.id) },
                    onDelete = { viewModel.deleteAlarm(alarm.id) }
                )
            }
        }
    }

    if (showTimePicker) {
        TimePickerDialog(
            onConfirm = { hour, minute ->
                viewModel.addAlarm(hour, minute)
                showTimePicker = false
            },
            onDismiss = { showTimePicker = false }
        )
    }
}

@Composable
fun AlarmItem(alarm: Alarm, onToggle: () -> Unit, onDelete: () -> Unit) {
    Card(Modifier.fillMaxWidth().padding(horizontal = 16.dp, vertical = 4.dp)) {
        Row(
            Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column(Modifier.weight(1f)) {
                Text(
                    "%02d:%02d".format(alarm.hour, alarm.minute),
                    style = MaterialTheme.typography.headlineMedium
                )
                Text(alarm.label, style = MaterialTheme.typography.bodySmall)
            }
            Switch(checked = alarm.isEnabled, onCheckedChange = { onToggle() })
        }
    }
}
```

---

## まとめ

| 種類 | 用途 | 精度 |
|------|------|------|
| `setExact` | 正確なアラーム | ±数秒 |
| `setRepeating` | 繰返し（非正確） | ±数分 |
| `WorkManager` | バックグラウンド定期処理 | 保証付き |

- `setExactAndAllowWhileIdle`でDoze中もアラーム発火
- Android 12+は`SCHEDULE_EXACT_ALARM`権限が必要
- `BroadcastReceiver`でアラーム受信し通知表示
- 繰り返しアラームは`WorkManager`推奨

---

8種類のAndroidアプリテンプレート（スケジューラー実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-notification-advanced-2026)
- [Foreground Service](https://zenn.dev/myougatheaxo/articles/android-compose-foreground-service-2026)
