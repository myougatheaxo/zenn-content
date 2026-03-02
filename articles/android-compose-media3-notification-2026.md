---
title: "Media3 Notification完全ガイド — メディア通知/MediaSession/バックグラウンド再生"
emoji: "🔔"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media3"]
published: true
---

## この記事で学べること

**Media3 Notification**（MediaSession、MediaNotification、バックグラウンド再生、メディアコントロール）を解説します。

---

## MediaSessionService

```kotlin
// build.gradle
// implementation "androidx.media3:media3-session:1.3.0"

class PlaybackService : MediaSessionService() {
    private var mediaSession: MediaSession? = null

    override fun onCreate() {
        super.onCreate()
        val player = ExoPlayer.Builder(this).build()
        mediaSession = MediaSession.Builder(this, player).build()
    }

    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo): MediaSession? =
        mediaSession

    override fun onDestroy() {
        mediaSession?.run {
            player.release()
            release()
        }
        super.onDestroy()
    }
}

// AndroidManifest.xml
// <service android:name=".PlaybackService"
//     android:foregroundServiceType="mediaPlayback"
//     android:exported="true">
//     <intent-filter>
//         <action android:name="androidx.media3.session.MediaSessionService" />
//     </intent-filter>
// </service>
```

---

## Compose接続

```kotlin
@Composable
fun MediaPlayerScreen() {
    val context = LocalContext.current
    var controller by remember { mutableStateOf<MediaController?>(null) }

    DisposableEffect(Unit) {
        val sessionToken = SessionToken(context, ComponentName(context, PlaybackService::class.java))
        val future = MediaController.Builder(context, sessionToken).buildAsync()

        future.addListener({
            controller = future.get()
        }, MoreExecutors.directExecutor())

        onDispose {
            controller?.release()
        }
    }

    controller?.let { ctrl ->
        PlayerControlUI(ctrl)
    } ?: Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        CircularProgressIndicator()
    }
}

@Composable
fun PlayerControlUI(controller: MediaController) {
    var isPlaying by remember { mutableStateOf(controller.isPlaying) }

    LaunchedEffect(controller) {
        val listener = object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) { isPlaying = playing }
        }
        controller.addListener(listener)
    }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        Text(controller.mediaMetadata.title?.toString() ?: "不明", style = MaterialTheme.typography.titleLarge)

        IconButton(onClick = { if (isPlaying) controller.pause() else controller.play() }) {
            Icon(
                if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                null, Modifier.size(64.dp)
            )
        }
    }
}
```

---

## メディアアイテム設定

```kotlin
fun createMediaItem(title: String, artist: String, uri: String): MediaItem =
    MediaItem.Builder()
        .setUri(uri)
        .setMediaMetadata(
            MediaMetadata.Builder()
                .setTitle(title)
                .setArtist(artist)
                .build()
        )
        .build()

// 使用
controller.setMediaItems(listOf(
    createMediaItem("曲1", "アーティスト1", "https://example.com/song1.mp3"),
    createMediaItem("曲2", "アーティスト2", "https://example.com/song2.mp3")
))
controller.prepare()
controller.play()
```

---

## まとめ

| API | 用途 |
|-----|------|
| `MediaSessionService` | バックグラウンド再生 |
| `MediaSession` | セッション管理 |
| `MediaController` | UI側制御 |
| `MediaMetadata` | メタ情報 |

- `MediaSessionService`でバックグラウンド再生+通知自動生成
- `MediaController`でCompose UIから制御
- メディア通知にはForeground Service Type `mediaPlayback`が必要
- `MediaMetadata`で曲名/アーティスト情報を設定

---

8種類のAndroidアプリテンプレート（メディア再生対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Media3 Player](https://zenn.dev/myougatheaxo/articles/android-compose-media3-player-2026)
- [Compose Notification](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
- [Compose Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-2026)
