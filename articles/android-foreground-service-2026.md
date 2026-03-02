---
title: "Foreground Service完全ガイド — バックグラウンドで動き続けるAndroidアプリ"
emoji: "⚙️"
type: "tech"
topics: ["android", "kotlin", "service", "notification"]
published: true
---

## この記事で学べること

Android 14対応の**Foreground Service**の実装方法を解説します。

---

## マニフェスト設定

```xml
<manifest>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application>
        <service
            android:name=".MusicService"
            android:foregroundServiceType="mediaPlayback"
            android:exported="false" />
    </application>
</manifest>
```

---

## 基本のForeground Service

```kotlin
class MusicService : Service() {

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START -> startForegroundService()
            ACTION_STOP -> stopSelf()
        }
        return START_STICKY
    }

    private fun startForegroundService() {
        val notification = createNotification()
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
            startForeground(
                NOTIFICATION_ID,
                notification,
                ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK
            )
        } else {
            startForeground(NOTIFICATION_ID, notification)
        }
    }

    private fun createNotification(): Notification {
        val channelId = "music_channel"
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                channelId, "音楽再生",
                NotificationManager.IMPORTANCE_LOW
            )
            getSystemService(NotificationManager::class.java)
                .createNotificationChannel(channel)
        }

        return NotificationCompat.Builder(this, channelId)
            .setContentTitle("再生中")
            .setContentText("楽曲名")
            .setSmallIcon(R.drawable.ic_music)
            .setOngoing(true)
            .build()
    }

    companion object {
        const val ACTION_START = "START"
        const val ACTION_STOP = "STOP"
        const val NOTIFICATION_ID = 1
    }
}
```

---

## Composeから制御

```kotlin
@Composable
fun MusicPlayerScreen() {
    val context = LocalContext.current
    var isPlaying by remember { mutableStateOf(false) }

    // 通知パーミッション（Android 13+）
    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) startService(context)
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        IconButton(
            onClick = {
                if (isPlaying) {
                    stopService(context)
                } else {
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                        permissionLauncher.launch(
                            Manifest.permission.POST_NOTIFICATIONS
                        )
                    } else {
                        startService(context)
                    }
                }
                isPlaying = !isPlaying
            }
        ) {
            Icon(
                if (isPlaying) Icons.Default.Stop else Icons.Default.PlayArrow,
                contentDescription = if (isPlaying) "停止" else "再生",
                modifier = Modifier.size(64.dp)
            )
        }
    }
}

private fun startService(context: Context) {
    val intent = Intent(context, MusicService::class.java).apply {
        action = MusicService.ACTION_START
    }
    ContextCompat.startForegroundService(context, intent)
}

private fun stopService(context: Context) {
    val intent = Intent(context, MusicService::class.java).apply {
        action = MusicService.ACTION_STOP
    }
    context.startService(intent)
}
```

---

## タイマー型Service

```kotlin
class TimerService : Service() {
    private val _elapsedSeconds = MutableStateFlow(0)
    val elapsedSeconds: StateFlow<Int> = _elapsedSeconds.asStateFlow()

    private var job: Job? = null

    override fun onBind(intent: Intent?): IBinder = TimerBinder()

    inner class TimerBinder : Binder() {
        fun getService(): TimerService = this@TimerService
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(1, createNotification(0))
        startTimer()
        return START_STICKY
    }

    private fun startTimer() {
        job = CoroutineScope(Dispatchers.Default).launch {
            while (isActive) {
                delay(1000)
                _elapsedSeconds.update { it + 1 }
                updateNotification(_elapsedSeconds.value)
            }
        }
    }

    private fun updateNotification(seconds: Int) {
        val notification = createNotification(seconds)
        getSystemService(NotificationManager::class.java)
            .notify(1, notification)
    }

    private fun createNotification(seconds: Int): Notification {
        val channelId = "timer_channel"
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                channelId, "タイマー",
                NotificationManager.IMPORTANCE_LOW
            )
            getSystemService(NotificationManager::class.java)
                .createNotificationChannel(channel)
        }

        val minutes = seconds / 60
        val secs = seconds % 60

        return NotificationCompat.Builder(this, channelId)
            .setContentTitle("タイマー実行中")
            .setContentText("%02d:%02d".format(minutes, secs))
            .setSmallIcon(R.drawable.ic_timer)
            .setOngoing(true)
            .build()
    }

    override fun onDestroy() {
        super.onDestroy()
        job?.cancel()
    }
}
```

---

## Android 14 のサービスタイプ

```kotlin
// Android 14以降は型指定が必須
// 主要なforegroundServiceType:
// - mediaPlayback: 音楽・動画再生
// - location: 位置情報取得
// - dataSync: データ同期
// - camera: カメラ使用
// - microphone: マイク使用
// - health: ヘルスケアデータ
// - shortService: 短時間処理（3分以内）

// マニフェストとstartForeground()の両方で指定
startForeground(
    NOTIFICATION_ID,
    notification,
    ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC
)
```

---

## まとめ

- `FOREGROUND_SERVICE`パーミッション + `foregroundServiceType`指定
- Android 13+は`POST_NOTIFICATIONS`パーミッション必須
- Android 14+は`startForeground()`に型パラメータ必須
- `START_STICKY`でプロセス復帰後も再起動
- 通知更新は`NotificationManager.notify()`
- Bindで双方向通信可能

---

8種類のAndroidアプリテンプレート（Service設計対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AlarmManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-alarm-scheduler-2026)
- [FCMプッシュ通知ガイド](https://zenn.dev/myougatheaxo/articles/android-fcm-push-notification-2026)
- [カウントダウンタイマー実装](https://zenn.dev/myougatheaxo/articles/android-compose-countdown-timer-2026)
