---
title: "Media3 + Compose完全ガイド — 動画再生/音楽プレーヤー/PiP"
emoji: "🎥"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media3"]
published: true
---

## この記事で学べること

**Media3 + Compose**（ExoPlayer、動画プレーヤーUI、音楽プレーヤー、PiP対応、メディア通知）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.media3:media3-exoplayer:1.5.1")
    implementation("androidx.media3:media3-ui-compose:1.5.1")
    implementation("androidx.media3:media3-session:1.5.1")
}
```

---

## 動画プレーヤー

```kotlin
@Composable
fun VideoPlayer(uri: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(uri))
            prepare()
        }
    }

    DisposableEffect(Unit) {
        onDispose { player.release() }
    }

    AndroidView(
        factory = { ctx ->
            PlayerView(ctx).apply {
                this.player = player
                useController = true
            }
        },
        modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
    )
}
```

---

## カスタムコントローラー

```kotlin
@Composable
fun CustomVideoPlayer(uri: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(uri))
            prepare()
        }
    }
    var isPlaying by remember { mutableStateOf(false) }
    var progress by remember { mutableFloatStateOf(0f) }
    var duration by remember { mutableLongStateOf(0L) }

    LaunchedEffect(player) {
        while (true) {
            isPlaying = player.isPlaying
            progress = if (player.duration > 0) player.currentPosition.toFloat() / player.duration else 0f
            duration = player.duration
            delay(200)
        }
    }

    DisposableEffect(Unit) { onDispose { player.release() } }

    Column {
        AndroidView(
            factory = { PlayerView(it).apply { this.player = player; useController = false } },
            modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        )

        Row(Modifier.fillMaxWidth().padding(8.dp), verticalAlignment = Alignment.CenterVertically) {
            IconButton(onClick = { if (isPlaying) player.pause() else player.play() }) {
                Icon(if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow, "再生/停止")
            }
            Slider(
                value = progress,
                onValueChange = { player.seekTo((it * duration).toLong()) },
                modifier = Modifier.weight(1f)
            )
            Text(formatTime(player.currentPosition), style = MaterialTheme.typography.labelSmall)
        }
    }
}

fun formatTime(ms: Long): String {
    val seconds = (ms / 1000) % 60
    val minutes = (ms / 1000 / 60) % 60
    return "%d:%02d".format(minutes, seconds)
}
```

---

## 音楽プレーヤー（MediaSession）

```kotlin
class MusicService : MediaSessionService() {
    private var mediaSession: MediaSession? = null

    override fun onCreate() {
        super.onCreate()
        val player = ExoPlayer.Builder(this).build()
        mediaSession = MediaSession.Builder(this, player).build()
    }

    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo) = mediaSession

    override fun onDestroy() {
        mediaSession?.run {
            player.release()
            release()
        }
        super.onDestroy()
    }
}
```

---

## PiP（ピクチャー・イン・ピクチャー）

```kotlin
@Composable
fun PipVideoPlayer(uri: String) {
    val context = LocalContext.current
    val activity = context as Activity

    Button(onClick = {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            activity.enterPictureInPictureMode(
                PictureInPictureParams.Builder()
                    .setAspectRatio(Rational(16, 9))
                    .build()
            )
        }
    }) { Text("PiPモード") }

    VideoPlayer(uri)
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 動画再生 | `ExoPlayer` + `PlayerView` |
| カスタムUI | `AndroidView` + Compose |
| 音楽サービス | `MediaSessionService` |
| PiP | `enterPictureInPictureMode` |
| 通知制御 | `MediaSession` |

- `ExoPlayer`で動画/音声再生
- `AndroidView`でPlayerViewをCompose統合
- `MediaSession`でメディア通知自動生成
- PiPでバックグラウンド視聴

---

8種類のAndroidアプリテンプレート（メディア再生対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [メディア再生](https://zenn.dev/myougatheaxo/articles/android-compose-media-player-2026)
- [フォアグラウンドサービス](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
- [Compose⇔View相互運用](https://zenn.dev/myougatheaxo/articles/android-compose-compose-interop-2026)
