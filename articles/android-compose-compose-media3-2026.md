---
title: "Compose Media3完全ガイド — ExoPlayer/MediaSession/オーディオ再生/バックグラウンド"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media3"]
published: true
---

## この記事で学べること

**Compose Media3**（ExoPlayer、MediaSession、動画/音声再生、バックグラウンド再生、Compose連携）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation("androidx.media3:media3-exoplayer:1.3.1")
    implementation("androidx.media3:media3-ui-compose:1.3.1")
    implementation("androidx.media3:media3-session:1.3.1")
}
```

---

## 動画プレーヤー

```kotlin
@Composable
fun VideoPlayer(url: String) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
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

## 音声プレーヤー

```kotlin
@HiltViewModel
class AudioViewModel @Inject constructor(
    @ApplicationContext private val context: Context
) : ViewModel() {
    val player = ExoPlayer.Builder(context).build()
    val isPlaying = MutableStateFlow(false)
    val progress = MutableStateFlow(0f)

    init {
        player.addListener(object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) {
                isPlaying.value = playing
            }
        })

        viewModelScope.launch {
            while (true) {
                if (player.isPlaying) {
                    progress.value = player.currentPosition.toFloat() / player.duration.coerceAtLeast(1)
                }
                delay(500)
            }
        }
    }

    fun play(url: String) {
        player.setMediaItem(MediaItem.fromUri(url))
        player.prepare()
        player.play()
    }

    fun togglePlayPause() {
        if (player.isPlaying) player.pause() else player.play()
    }

    override fun onCleared() { player.release() }
}

@Composable
fun AudioPlayerUI(viewModel: AudioViewModel = hiltViewModel()) {
    val isPlaying by viewModel.isPlaying.collectAsStateWithLifecycle()
    val progress by viewModel.progress.collectAsStateWithLifecycle()

    Card(Modifier.fillMaxWidth().padding(16.dp)) {
        Column(Modifier.padding(16.dp)) {
            Text("オーディオプレーヤー", style = MaterialTheme.typography.titleMedium)
            Spacer(Modifier.height(8.dp))
            LinearProgressIndicator(progress = { progress }, modifier = Modifier.fillMaxWidth())
            Spacer(Modifier.height(8.dp))
            Row(horizontalArrangement = Arrangement.Center, modifier = Modifier.fillMaxWidth()) {
                IconButton(onClick = { viewModel.player.seekBack() }) {
                    Icon(Icons.Default.Replay10, "10秒戻る")
                }
                IconButton(onClick = { viewModel.togglePlayPause() }) {
                    Icon(
                        if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                        if (isPlaying) "一時停止" else "再生"
                    )
                }
                IconButton(onClick = { viewModel.player.seekForward() }) {
                    Icon(Icons.Default.Forward10, "10秒進む")
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ExoPlayer` | メディア再生 |
| `MediaSession` | セッション管理 |
| `PlayerView` | 動画UI |
| `MediaItem` | メディアソース |

- Media3はExoPlayerの後継ライブラリ
- `PlayerView`をAndroidViewでComposeに埋め込み
- `MediaSession`でバックグラウンド再生と通知制御
- プレイリスト対応、DRM対応

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ExoPlayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-exoplayer-2026)
- [Compose Audio](https://zenn.dev/myougatheaxo/articles/android-compose-compose-audio-2026)
- [Compose CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-compose-camerax-2026)
