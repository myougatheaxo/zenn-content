---
title: "Compose ExoPlayer完全ガイド — カスタムUI/プレイリスト/DRM/ピクチャーインピクチャー"
emoji: "▶️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "exoplayer"]
published: true
---

## この記事で学べること

**Compose ExoPlayer**（カスタムコントロール、プレイリスト、DRM対応、PiP）を解説します。

---

## カスタムコントロール

```kotlin
@Composable
fun CustomVideoPlayer(url: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
            prepare()
        }
    }
    var isPlaying by remember { mutableStateOf(false) }
    var showControls by remember { mutableStateOf(true) }

    DisposableEffect(Unit) {
        val listener = object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) { isPlaying = playing }
        }
        player.addListener(listener)
        onDispose { player.removeListener(listener); player.release() }
    }

    Box(Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        .pointerInput(Unit) { detectTapGestures { showControls = !showControls } }
    ) {
        AndroidView(factory = { PlayerView(it).apply { this.player = player; useController = false } },
            modifier = Modifier.fillMaxSize())

        AnimatedVisibility(showControls, modifier = Modifier.align(Alignment.Center)) {
            IconButton(
                onClick = { if (isPlaying) player.pause() else player.play() },
                modifier = Modifier.size(64.dp).background(Color.Black.copy(0.5f), CircleShape)
            ) {
                Icon(
                    if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                    contentDescription = null, tint = Color.White, modifier = Modifier.size(40.dp)
                )
            }
        }
    }
}
```

---

## プレイリスト

```kotlin
@Composable
fun PlaylistPlayer(videos: List<VideoItem>) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            videos.forEach { addMediaItem(MediaItem.fromUri(it.url)) }
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
        onDispose { player.release() }
    }

    Column(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { PlayerView(it).apply { this.player = player } },
            modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        )

        LazyColumn {
            itemsIndexed(videos) { index, video ->
                ListItem(
                    headlineContent = { Text(video.title) },
                    leadingContent = {
                        if (index == currentIndex)
                            Icon(Icons.Default.PlayArrow, null, tint = MaterialTheme.colorScheme.primary)
                    },
                    modifier = Modifier.clickable { player.seekTo(index, 0) }
                )
            }
        }
    }
}
```

---

## PiP対応

```kotlin
@Composable
fun PipVideoPlayer(url: String) {
    val context = LocalContext.current
    val player = remember { ExoPlayer.Builder(context).build() }

    LaunchedEffect(url) {
        player.setMediaItem(MediaItem.fromUri(url))
        player.prepare()
        player.play()
    }

    DisposableEffect(Unit) { onDispose { player.release() } }

    // PiPモードに入る
    val activity = context as Activity
    fun enterPip() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            activity.enterPictureInPictureMode(
                PictureInPictureParams.Builder()
                    .setAspectRatio(Rational(16, 9))
                    .build()
            )
        }
    }

    Column(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { PlayerView(it).apply { this.player = player } },
            modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        )
        Button(onClick = { enterPip() }, modifier = Modifier.padding(16.dp)) {
            Text("ピクチャーインピクチャー")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ExoPlayer` | 動画再生エンジン |
| `PlayerView` | 標準UI |
| `addMediaItem` | プレイリスト |
| `PictureInPictureParams` | PiP |

- カスタムコントロールは`useController = false`で自作
- `addMediaItem`でプレイリスト対応
- PiPで小窓再生
- `Player.Listener`で再生状態をCompose UIに反映

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Media3](https://zenn.dev/myougatheaxo/articles/android-compose-compose-media3-2026)
- [Compose Audio](https://zenn.dev/myougatheaxo/articles/android-compose-compose-audio-2026)
- [Compose ImmersiveMode](https://zenn.dev/myougatheaxo/articles/android-compose-compose-immersive-mode-2026)
