---
title: "Media3/ExoPlayer完全ガイド — 動画再生/プレイリスト/PiP/バックグラウンド"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media"]
published: true
---

## この記事で学べること

**Media3/ExoPlayer**（動画再生、プレイリスト管理、Picture-in-Picture、バックグラウンド再生）を解説します。

---

## 基本動画プレーヤー

```kotlin
@Composable
fun VideoPlayer(uri: Uri, modifier: Modifier = Modifier) {
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
        modifier = modifier
    )
}
```

---

## プレイリスト

```kotlin
@Composable
fun PlaylistPlayer(items: List<MediaItem>) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItems(items)
            prepare()
        }
    }

    var currentIndex by remember { mutableIntStateOf(0) }

    DisposableEffect(Unit) {
        val listener = object : Player.Listener {
            override fun onMediaItemTransition(mediaItem: MediaItem?, reason: Int) {
                currentIndex = player.currentMediaItemIndex
            }
        }
        player.addListener(listener)
        onDispose {
            player.removeListener(listener)
            player.release()
        }
    }

    Column {
        AndroidView(
            factory = { PlayerView(it).apply { this.player = player } },
            modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        )

        LazyColumn {
            itemsIndexed(items) { index, item ->
                ListItem(
                    headlineContent = { Text(item.mediaMetadata.title?.toString() ?: "Track $index") },
                    leadingContent = {
                        if (index == currentIndex) Icon(Icons.Default.PlayArrow, null, tint = MaterialTheme.colorScheme.primary)
                        else Text("${index + 1}")
                    },
                    modifier = Modifier.clickable { player.seekTo(index, 0) }
                )
            }
        }
    }
}
```

---

## MediaSessionService（バックグラウンド）

```kotlin
class PlaybackService : MediaSessionService() {
    private var mediaSession: MediaSession? = null

    override fun onCreate() {
        super.onCreate()
        val player = ExoPlayer.Builder(this).build()
        mediaSession = MediaSession.Builder(this, player).build()
    }

    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo): MediaSession? {
        return mediaSession
    }

    override fun onDestroy() {
        mediaSession?.run {
            player.release()
            release()
        }
        super.onDestroy()
    }
}

// Compose側で接続
@Composable
fun RemotePlayerUI() {
    val context = LocalContext.current
    val controllerFuture = remember {
        MediaController.Builder(
            context,
            SessionToken(context, ComponentName(context, PlaybackService::class.java))
        ).buildAsync()
    }

    var isPlaying by remember { mutableStateOf(false) }

    DisposableEffect(Unit) {
        onDispose { MediaController.releaseFuture(controllerFuture) }
    }

    Row(horizontalArrangement = Arrangement.Center, modifier = Modifier.fillMaxWidth()) {
        IconButton(onClick = {
            controllerFuture.get()?.let {
                if (it.isPlaying) it.pause() else it.play()
            }
        }) {
            Icon(if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow, "再生/一時停止")
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 動画再生 | `ExoPlayer` + `PlayerView` |
| プレイリスト | `setMediaItems` |
| バックグラウンド | `MediaSessionService` |
| リモート制御 | `MediaController` |

- `ExoPlayer`で高品質な動画再生
- `MediaItem`リストでプレイリスト管理
- `MediaSessionService`でバックグラウンド再生
- `PlayerView`をAndroidViewでCompose統合

---

8種類のAndroidアプリテンプレート（メディア再生対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [音声録音](https://zenn.dev/myougatheaxo/articles/android-compose-audio-recorder-2026)
- [動画録画](https://zenn.dev/myougatheaxo/articles/android-compose-video-recorder-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-2026)
