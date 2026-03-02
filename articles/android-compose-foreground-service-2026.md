---
title: "フォアグラウンドサービス完全ガイド — 通知/位置情報/音楽再生/タイマー"
emoji: "🔧"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "service"]
published: true
---

## この記事で学べること

**フォアグラウンドサービス**（通知表示、位置情報追跡、音楽再生、タイマーサービス）を解説します。

---

## 基本フォアグラウンドサービス

```kotlin
class TimerService : Service() {
    private val binder = LocalBinder()
    private var timerJob: Job? = null
    private val _elapsed = MutableStateFlow(0L)
    val elapsed: StateFlow<Long> = _elapsed.asStateFlow()

    inner class LocalBinder : Binder() {
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
        val notification = NotificationCompat.Builder(this, "timer_channel")
            .setContentTitle("タイマー実行中")
            .setSmallIcon(R.drawable.ic_timer)
            .setOngoing(true)
            .build()

        startForeground(1, notification)

        timerJob = CoroutineScope(Dispatchers.Default).launch {
            while (isActive) {
                delay(1000)
                _elapsed.update { it + 1 }
                updateNotification(_elapsed.value)
            }
        }
    }

    private fun stopTimer() {
        timerJob?.cancel()
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
    }
}
```

---

## Manifest宣言

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />

<service
    android:name=".TimerService"
    android:foregroundServiceType="specialUse"
    android:exported="false" />
```

---

## Compose連携

```kotlin
@Composable
fun TimerScreen() {
    val context = LocalContext.current
    var elapsed by remember { mutableLongStateOf(0L) }
    var isRunning by remember { mutableStateOf(false) }

    val connection = remember {
        object : ServiceConnection {
            var service: TimerService? = null

            override fun onServiceConnected(name: ComponentName, binder: IBinder) {
                service = (binder as TimerService.LocalBinder).getService()
            }
            override fun onServiceDisconnected(name: ComponentName) {
                service = null
            }
        }
    }

    LaunchedEffect(isRunning) {
        connection.service?.elapsed?.collect { elapsed = it }
    }

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            formatTime(elapsed),
            style = MaterialTheme.typography.displayLarge
        )
        Spacer(Modifier.height(32.dp))
        Button(onClick = {
            isRunning = !isRunning
            val intent = Intent(context, TimerService::class.java).apply {
                action = if (isRunning) "START" else "STOP"
            }
            context.startForegroundService(intent)
        }) {
            Text(if (isRunning) "停止" else "開始")
        }
    }
}
```

---

## まとめ

| サービスタイプ | foregroundServiceType |
|---------------|----------------------|
| 位置情報 | `location` |
| 音楽再生 | `mediaPlayback` |
| カメラ | `camera` |
| 汎用 | `specialUse` |

- `startForeground`で通知付きサービス起動
- `foregroundServiceType`をManifestで宣言必須
- `ServiceConnection`でComposeとサービス間通信
- `StateFlow`でリアクティブなUI更新

---

8種類のAndroidアプリテンプレート（サービス対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [AlarmManager](https://zenn.dev/myougatheaxo/articles/android-compose-alarm-scheduler-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-2026)
