---
title: "Compose ForegroundService完全ガイド — フォアグラウンドサービス/通知/タイマー"
emoji: "▶️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "service"]
published: true
---

## この記事で学べること

**Compose ForegroundService**（フォアグラウンドサービス、通知表示、タイマー、Compose連携）を解説します。

---

## 基本ForegroundService

```kotlin
class TimerService : Service() {
    private var seconds = 0
    private var isRunning = false
    private val binder = TimerBinder()

    inner class TimerBinder : Binder() {
        fun getService(): TimerService = this@TimerService
    }

    override fun onBind(intent: Intent): IBinder = binder

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            "START" -> startTimer()
            "STOP" -> stopTimer()
        }
        return START_STICKY
    }

    private fun startTimer() {
        isRunning = true
        val notification = createNotification()
        startForeground(1, notification)

        CoroutineScope(Dispatchers.Default).launch {
            while (isRunning) {
                delay(1000)
                seconds++
                updateNotification()
            }
        }
    }

    private fun createNotification(): Notification {
        val channelId = "timer_channel"
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(channelId, "タイマー", NotificationManager.IMPORTANCE_LOW)
            getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
        }
        return NotificationCompat.Builder(this, channelId)
            .setSmallIcon(R.drawable.ic_timer)
            .setContentTitle("タイマー実行中")
            .setContentText(formatTime(seconds))
            .setOngoing(true)
            .build()
    }

    private fun formatTime(s: Int) = "%02d:%02d".format(s / 60, s % 60)
    private fun stopTimer() { isRunning = false; stopForeground(STOP_FOREGROUND_REMOVE); stopSelf() }
    private fun updateNotification() { /* 通知更新 */ }
}
```

---

## Compose連携

```kotlin
@Composable
fun TimerScreen() {
    val context = LocalContext.current
    var isRunning by remember { mutableStateOf(false) }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        Button(onClick = {
            val intent = Intent(context, TimerService::class.java).apply {
                action = if (isRunning) "STOP" else "START"
            }
            if (!isRunning) context.startForegroundService(intent)
            else context.startService(intent)
            isRunning = !isRunning
        }, Modifier.fillMaxWidth()) {
            Text(if (isRunning) "停止" else "開始")
        }
    }
}
```

---

## AndroidManifest

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service
    android:name=".TimerService"
    android:foregroundServiceType="specialUse"
    android:exported="false" />
```

---

## まとめ

| API | 用途 |
|-----|------|
| `startForeground` | フォアグラウンド開始 |
| `NotificationChannel` | 通知チャネル |
| `START_STICKY` | サービス再起動 |
| `foregroundServiceType` | サービス種別 |

- Android 14+は`foregroundServiceType`指定必須
- `POST_NOTIFICATIONS`パーミッション（Android 13+）
- `startForeground`は5秒以内に呼ぶ必要あり
- `stopForeground`+`stopSelf`で適切に終了

---

8種類のAndroidアプリテンプレート（バックグラウンド処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager Basic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-basic-2026)
- [Compose Notification](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
- [Compose BatteryState](https://zenn.dev/myougatheaxo/articles/android-compose-compose-battery-state-2026)
