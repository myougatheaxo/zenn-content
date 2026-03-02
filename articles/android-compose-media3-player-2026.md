---
title: "Media3 Player完全ガイド — ExoPlayer/Compose統合/メディア再生"
emoji: "▶️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media3"]
published: true
---

## この記事で学べること

**Media3 Player**（ExoPlayer、Compose統合、音声/動画再生、プレイリスト管理）を解説します。

---

## 基本セットアップ

```kotlin
// build.gradle
// implementation "androidx.media3:media3-exoplayer:1.3.0"
// implementation "androidx.media3:media3-ui:1.3.0"

@Composable
fun VideoPlayerScreen(videoUrl: String) {
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
        modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
    )
}
```

---

## カスタムコントロール

```kotlin
@Composable
fun CustomPlayerControls(player: ExoPlayer) {
    var isPlaying by remember { mutableStateOf(player.isPlaying) }
    var progress by remember { mutableFloatStateOf(0f) }

    LaunchedEffect(player) {
        while (true) {
            isPlaying = player.isPlaying
            progress = if (player.duration > 0)
                player.currentPosition.toFloat() / player.duration
            else 0f
            delay(500)
        }
    }

    Column(Modifier.padding(16.dp)) {
        Slider(
            value = progress,
            onValueChange = {
                player.seekTo((it * player.duration).toLong())
            }
        )

        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.Center,
            verticalAlignment = Alignment.CenterVertically
        ) {
            IconButton(onClick = { player.seekBack() }) {
                Icon(Icons.Default.Replay10, "10秒戻る")
            }
            IconButton(onClick = {
                if (isPlaying) player.pause() else player.play()
            }) {
                Icon(
                    if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                    if (isPlaying) "一時停止" else "再生",
                    modifier = Modifier.size(48.dp)
                )
            }
            IconButton(onClick = { player.seekForward() }) {
                Icon(Icons.Default.Forward10, "10秒進む")
            }
        }

        Text(
            "${formatTime(player.currentPosition)} / ${formatTime(player.duration)}",
            style = MaterialTheme.typography.bodySmall,
            modifier = Modifier.align(Alignment.CenterHorizontally)
        )
    }
}

fun formatTime(ms: Long): String {
    val seconds = (ms / 1000) % 60
    val minutes = (ms / 1000 / 60) % 60
    return "%02d:%02d".format(minutes, seconds)
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
            val items = urls.map { MediaItem.fromUri(it) }
            setMediaItems(items)
            prepare()
        }
    }

    DisposableEffect(Unit) { onDispose { player.release() } }

    Column {
        AndroidView(
            factory = { PlayerView(it).apply { this.player = player } },
            modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        )
        LazyColumn {
            itemsIndexed(urls) { index, url ->
                ListItem(
                    headlineContent = { Text("トラック ${index + 1}") },
                    modifier = Modifier.clickable { player.seekToDefaultPosition(index) }
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ExoPlayer.Builder` | プレイヤー構築 |
| `PlayerView` | 標準UI |
| `MediaItem` | メディアアイテム |
| `setMediaItems` | プレイリスト |

- `DisposableEffect`で必ず`player.release()`
- `AndroidView`でPlayerViewをCompose内に配置
- カスタムコントロールで独自UIを構築可能
- プレイリスト管理で連続再生

---

8種類のAndroidアプリテンプレート（メディア再生対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Media3 Notification](https://zenn.dev/myougatheaxo/articles/android-compose-media3-notification-2026)
- [Compose VideoPlayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-video-player-2026)
- [Compose Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-2026)
