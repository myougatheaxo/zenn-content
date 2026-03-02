---
title: "Video Player完全ガイド — ExoPlayer/Media3/カスタムコントロール"
emoji: "🎥"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media3"]
published: true
---

## この記事で学べること

**Video Player**（Media3 ExoPlayer、AndroidView、カスタムコントロール）を解説します。

---

## ExoPlayer基本

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
        modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
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
    var progress by remember { mutableFloatStateOf(0f) }
    var duration by remember { mutableLongStateOf(0L) }

    LaunchedEffect(player) {
        while (true) {
            delay(500)
            if (player.duration > 0) {
                progress = player.currentPosition.toFloat() / player.duration
                duration = player.duration
            }
            isPlaying = player.isPlaying
        }
    }

    DisposableEffect(Unit) { onDispose { player.release() } }

    Column {
        Box {
            AndroidView(
                factory = { PlayerView(it).apply { this.player = player; useController = false } },
                modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
            )
        }

        // コントロール
        Row(Modifier.fillMaxWidth().padding(8.dp), verticalAlignment = Alignment.CenterVertically) {
            IconButton(onClick = {
                if (isPlaying) player.pause() else player.play()
            }) {
                Icon(if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow, "再生/停止")
            }

            Slider(
                value = progress,
                onValueChange = { player.seekTo((it * player.duration).toLong()) },
                modifier = Modifier.weight(1f)
            )

            Text(
                formatDuration(player.currentPosition) + "/" + formatDuration(duration),
                style = MaterialTheme.typography.bodySmall
            )
        }
    }
}

fun formatDuration(ms: Long): String {
    val minutes = (ms / 1000) / 60
    val seconds = (ms / 1000) % 60
    return "%d:%02d".format(minutes, seconds)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ExoPlayer` | 動画再生エンジン |
| `PlayerView` | 標準プレーヤーUI |
| `MediaItem` | メディアソース |
| `AndroidView` | Compose内に配置 |

- Media3 ExoPlayerで高品質な動画再生
- `AndroidView`でCompose内にPlayerView配置
- カスタムコントロールで独自UIを実装
- `DisposableEffect`でリソース解放を保証

---

8種類のAndroidアプリテンプレート（メディア対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カメラプレビュー](https://zenn.dev/myougatheaxo/articles/android-compose-compose-camera-preview-2026)
- [Slider](https://zenn.dev/myougatheaxo/articles/android-compose-compose-slider-2026)
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
