---
title: "Media3 ExoPlayer + Compose — 動画/音声再生ガイド"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media"]
published: true
---

## この記事で学べること

**Media3 ExoPlayer**をComposeで使う方法（動画再生、音声再生、プレイリスト、バックグラウンド再生）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.media3:media3-exoplayer:1.4.1")
    implementation("androidx.media3:media3-ui-compose:1.4.1")
    implementation("androidx.media3:media3-session:1.4.1")
}
```

---

## 動画プレーヤー

```kotlin
@Composable
fun VideoPlayer(url: String, modifier: Modifier = Modifier) {
    val context = LocalContext.current

    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
            prepare()
            playWhenReady = true
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
        modifier = modifier.fillMaxWidth().aspectRatio(16f / 9f)
    )
}
```

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
    var currentPosition by remember { mutableLongStateOf(0L) }
    var duration by remember { mutableLongStateOf(0L) }

    DisposableEffect(player) {
        val listener = object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) {
                isPlaying = playing
            }
            override fun onPlaybackStateChanged(state: Int) {
                if (state == Player.STATE_READY) {
                    duration = player.duration
                }
            }
        }
        player.addListener(listener)
        onDispose {
            player.removeListener(listener)
            player.release()
        }
    }

    // 位置更新
    LaunchedEffect(isPlaying) {
        while (isPlaying) {
            currentPosition = player.currentPosition
            delay(500)
        }
    }

    Column {
        AndroidView(
            factory = { PlayerView(it).apply { this.player = player; useController = false } },
            modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
        )

        // スライダー
        Slider(
            value = currentPosition.toFloat(),
            onValueChange = { player.seekTo(it.toLong()) },
            valueRange = 0f..duration.toFloat()
        )

        // コントロールボタン
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
                    "再生/一時停止"
                )
            }
            IconButton(onClick = { player.seekForward() }) {
                Icon(Icons.Default.Forward10, "10秒進む")
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
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(audioUrl))
            prepare()
        }
    }

    var isPlaying by remember { mutableStateOf(false) }

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

    Row(
        Modifier.fillMaxWidth().padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        IconButton(onClick = {
            if (isPlaying) player.pause() else player.play()
        }) {
            Icon(
                if (isPlaying) Icons.Default.PauseCircle else Icons.Default.PlayCircle,
                null,
                modifier = Modifier.size(48.dp),
                tint = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

---

## プレイリスト

```kotlin
@Composable
fun PlaylistPlayer(tracks: List<Track>) {
    val context = LocalContext.current

    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            val mediaItems = tracks.map { MediaItem.fromUri(it.url) }
            setMediaItems(mediaItems)
            prepare()
        }
    }

    var currentIndex by remember { mutableIntStateOf(0) }

    DisposableEffect(player) {
        val listener = object : Player.Listener {
            override fun onMediaItemTransition(mediaItem: MediaItem?, reason: Int) {
                currentIndex = player.currentMediaItemIndex
            }
        }
        player.addListener(listener)
        onDispose { player.removeListener(listener); player.release() }
    }

    Column {
        // 現在のトラック
        Text(
            tracks.getOrNull(currentIndex)?.title ?: "",
            style = MaterialTheme.typography.titleLarge
        )

        // トラックリスト
        LazyColumn {
            itemsIndexed(tracks) { index, track ->
                ListItem(
                    headlineContent = { Text(track.title) },
                    leadingContent = {
                        if (index == currentIndex) {
                            Icon(Icons.Default.MusicNote, null, tint = MaterialTheme.colorScheme.primary)
                        }
                    },
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

## まとめ

- `ExoPlayer.Builder(context).build()`でプレーヤー作成
- `PlayerView`を`AndroidView`でCompose統合
- `Player.Listener`で状態変更を監視
- `DisposableEffect`の`onDispose`で`player.release()`必須
- `setMediaItems`でプレイリスト設定
- `seekTo(index, 0)`でトラック切り替え

---

8種類のAndroidアプリテンプレート（メディア再生対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Media3 ExoPlayer](https://zenn.dev/myougatheaxo/articles/android-media3-exoplayer-2026)
- [Composeライフサイクル](https://zenn.dev/myougatheaxo/articles/android-compose-lifecycle-aware-2026)
- [バックグラウンドサービス](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
