---
title: "Media3 (ExoPlayer) + Compose — 動画・音声プレーヤーの実装"
emoji: "▶️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media"]
published: true
---

## この記事で学べること

**Media3 (ExoPlayer)**をComposeアプリに統合し、動画・音声プレーヤーを実装する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.media3:media3-exoplayer:1.4.0")
    implementation("androidx.media3:media3-ui-compose:1.4.0")
    implementation("androidx.media3:media3-ui:1.4.0")
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
            }
        },
        modifier = Modifier
            .fillMaxWidth()
            .aspectRatio(16f / 9f)
    )
}
```

---

## プレイリスト

```kotlin
@Composable
fun PlaylistPlayer(urls: List<String>) {
    val context = LocalContext.current

    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            val mediaItems = urls.map { MediaItem.fromUri(it) }
            setMediaItems(mediaItems)
            prepare()
        }
    }

    DisposableEffect(Unit) {
        onDispose { player.release() }
    }

    Column {
        AndroidView(
            factory = { ctx -> PlayerView(ctx).apply { this.player = player } },
            modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        )

        // プレイリスト表示
        LazyColumn {
            itemsIndexed(urls) { index, url ->
                ListItem(
                    headlineContent = { Text("トラック ${index + 1}") },
                    modifier = Modifier.clickable {
                        player.seekTo(index, 0)
                        player.play()
                    }
                )
            }
        }
    }
}
```

---

## 音声プレーヤー

```kotlin
@Composable
fun AudioPlayer(audioUrl: String) {
    val context = LocalContext.current
    var isPlaying by remember { mutableStateOf(false) }
    var progress by remember { mutableFloatStateOf(0f) }

    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(audioUrl))
            prepare()
        }
    }

    LaunchedEffect(player) {
        while (true) {
            if (player.isPlaying) {
                progress = player.currentPosition.toFloat() / player.duration.coerceAtLeast(1)
            }
            isPlaying = player.isPlaying
            delay(500)
        }
    }

    DisposableEffect(Unit) {
        onDispose { player.release() }
    }

    Card(Modifier.fillMaxWidth().padding(16.dp)) {
        Column(Modifier.padding(16.dp)) {
            Text("音声プレーヤー", style = MaterialTheme.typography.titleMedium)
            Spacer(Modifier.height(8.dp))

            LinearProgressIndicator(
                progress = { progress },
                modifier = Modifier.fillMaxWidth()
            )
            Spacer(Modifier.height(8.dp))

            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.Center
            ) {
                IconButton(onClick = { player.seekBack() }) {
                    Icon(Icons.Default.Replay10, "10秒戻る")
                }
                IconButton(onClick = {
                    if (isPlaying) player.pause() else player.play()
                }) {
                    Icon(
                        if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                        if (isPlaying) "一時停止" else "再生"
                    )
                }
                IconButton(onClick = { player.seekForward() }) {
                    Icon(Icons.Default.Forward10, "10秒進む")
                }
            }
        }
    }
}
```

---

## ライフサイクル対応

```kotlin
@Composable
fun LifecycleAwarePlayer(url: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
            prepare()
        }
    }

    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_PAUSE -> player.pause()
                Lifecycle.Event.ON_RESUME -> player.play()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
            player.release()
        }
    }

    AndroidView(
        factory = { ctx -> PlayerView(ctx).apply { this.player = player } },
        modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
    )
}
```

---

## まとめ

- `ExoPlayer.Builder` + `MediaItem`で再生
- `PlayerView` + `AndroidView`でCompose統合
- `DisposableEffect`で`player.release()`を忘れずに
- `LifecycleEventObserver`でバックグラウンド時にpause
- プレイリスト: `setMediaItems` + `seekTo(index)`
- 音声: カスタムUIでplay/pause/seek

---

8種類のAndroidアプリテンプレート（メディア再生機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX実装ガイド](https://zenn.dev/myougatheaxo/articles/android-camerax-compose-2026)
- [Lifecycle完全ガイド](https://zenn.dev/myougatheaxo/articles/android-lifecycle-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
