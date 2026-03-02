---
title: "Media3/ExoPlayer Composeガイド — 動画/音声プレーヤー実装"
emoji: "▶️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media"]
published: true
---

## この記事で学べること

**Media3（ExoPlayer）+ Compose**での動画/音声プレーヤー実装を解説します。

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

## 基本の動画プレーヤー

```kotlin
@Composable
fun VideoPlayer(videoUrl: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(videoUrl))
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
        modifier = Modifier
            .fillMaxWidth()
            .aspectRatio(16f / 9f)
    )
}
```

---

## カスタムコントロール

```kotlin
@Composable
fun CustomVideoPlayer(videoUrl: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(videoUrl))
            prepare()
        }
    }
    var isPlaying by remember { mutableStateOf(false) }
    var currentPosition by remember { mutableLongStateOf(0L) }
    var duration by remember { mutableLongStateOf(0L) }

    DisposableEffect(player) {
        val listener = object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) {
                isPlaying = playing
            }
        }
        player.addListener(listener)
        onDispose {
            player.removeListener(listener)
            player.release()
        }
    }

    // 再生位置の定期更新
    LaunchedEffect(isPlaying) {
        while (isPlaying) {
            currentPosition = player.currentPosition
            duration = player.duration.coerceAtLeast(1L)
            delay(500)
        }
    }

    Column {
        AndroidView(
            factory = { ctx ->
                PlayerView(ctx).apply {
                    this.player = player
                    useController = false // カスタムコントロール使用
                }
            },
            modifier = Modifier
                .fillMaxWidth()
                .aspectRatio(16f / 9f)
        )

        // カスタムコントロール
        Column(Modifier.padding(8.dp)) {
            Slider(
                value = if (duration > 0) currentPosition.toFloat() / duration else 0f,
                onValueChange = { player.seekTo((it * duration).toLong()) }
            )
            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(formatDuration(currentPosition))
                IconButton(onClick = {
                    if (isPlaying) player.pause() else player.play()
                }) {
                    Icon(
                        if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                        "再生/一時停止"
                    )
                }
                Text(formatDuration(duration))
            }
        }
    }
}

fun formatDuration(ms: Long): String {
    val seconds = (ms / 1000) % 60
    val minutes = (ms / 1000 / 60) % 60
    return "%d:%02d".format(minutes, seconds)
}
```

---

## 音声プレーヤー

```kotlin
@Composable
fun AudioPlayer(audioUrl: String, title: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(audioUrl))
            prepare()
        }
    }
    var isPlaying by remember { mutableStateOf(false) }

    DisposableEffect(player) {
        val listener = object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) { isPlaying = playing }
        }
        player.addListener(listener)
        onDispose { player.removeListener(listener); player.release() }
    }

    Card(Modifier.fillMaxWidth()) {
        Row(
            Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            IconButton(onClick = {
                if (isPlaying) player.pause() else player.play()
            }) {
                Icon(
                    if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                    "再生"
                )
            }
            Column(Modifier.weight(1f)) {
                Text(title, style = MaterialTheme.typography.titleSmall)
                Text("音声ファイル", style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}
```

---

## まとめ

- `ExoPlayer.Builder(context).build()`でプレーヤー作成
- `AndroidView { PlayerView }`でCompose内に表示
- `DisposableEffect`で`player.release()`を確実に呼ぶ
- `Player.Listener`で再生状態を監視
- カスタムコントロールで`useController = false`
- `LaunchedEffect`で再生位置を定期更新

---

8種類のAndroidアプリテンプレート（メディア再生対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Media3/ExoPlayerガイド](https://zenn.dev/myougatheaxo/articles/android-media3-exoplayer-2026)
- [View Interopガイド](https://zenn.dev/myougatheaxo/articles/compose-view-interop-2026)
- [Foreground Serviceガイド](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
