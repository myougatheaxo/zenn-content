---
title: "Compose Audio完全ガイド — MediaPlayer/AudioFocus/録音/波形表示"
emoji: "🔊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "audio"]
published: true
---

## この記事で学べること

**Compose Audio**（MediaPlayer、Audio Focus、録音、波形表示、Compose連携）を解説します。

---

## 音声再生

```kotlin
@HiltViewModel
class AudioPlayerViewModel @Inject constructor(
    @ApplicationContext private val context: Context
) : ViewModel() {
    private var mediaPlayer: MediaPlayer? = null
    var isPlaying by mutableStateOf(false)
        private set
    var duration by mutableIntStateOf(0)
        private set
    var currentPosition by mutableIntStateOf(0)
        private set

    fun play(resId: Int) {
        mediaPlayer?.release()
        mediaPlayer = MediaPlayer.create(context, resId).apply {
            setOnCompletionListener { isPlaying = false }
            start()
        }
        isPlaying = true
        duration = mediaPlayer?.duration ?: 0
        updatePosition()
    }

    private fun updatePosition() {
        viewModelScope.launch {
            while (isPlaying) {
                currentPosition = mediaPlayer?.currentPosition ?: 0
                delay(100)
            }
        }
    }

    fun togglePlayPause() {
        mediaPlayer?.let {
            if (it.isPlaying) { it.pause(); isPlaying = false }
            else { it.start(); isPlaying = true; updatePosition() }
        }
    }

    fun seekTo(position: Int) { mediaPlayer?.seekTo(position) }
    override fun onCleared() { mediaPlayer?.release() }
}
```

---

## 録音

```kotlin
class AudioRecorder @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private var recorder: MediaRecorder? = null
    var isRecording by mutableStateOf(false)
        private set

    @SuppressLint("MissingPermission")
    fun startRecording(outputFile: File) {
        recorder = MediaRecorder(context).apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setAudioSamplingRate(44100)
            setAudioEncodingBitRate(128000)
            setOutputFile(outputFile.absolutePath)
            prepare()
            start()
        }
        isRecording = true
    }

    fun stopRecording() {
        recorder?.apply { stop(); release() }
        recorder = null
        isRecording = false
    }
}

@Composable
fun RecorderUI(recorder: AudioRecorder) {
    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        IconButton(
            onClick = {
                if (recorder.isRecording) recorder.stopRecording()
                else recorder.startRecording(File.createTempFile("rec", ".m4a"))
            },
            modifier = Modifier.size(80.dp)
                .background(if (recorder.isRecording) Color.Red else MaterialTheme.colorScheme.primary, CircleShape)
        ) {
            Icon(
                if (recorder.isRecording) Icons.Default.Stop else Icons.Default.Mic,
                contentDescription = null, tint = Color.White, modifier = Modifier.size(40.dp)
            )
        }
        Spacer(Modifier.height(16.dp))
        Text(if (recorder.isRecording) "録音中..." else "タップで録音開始")
    }
}
```

---

## 波形表示

```kotlin
@Composable
fun WaveformView(amplitudes: List<Float>, modifier: Modifier = Modifier) {
    Canvas(modifier.fillMaxWidth().height(100.dp)) {
        val barWidth = size.width / amplitudes.size.coerceAtLeast(1)
        amplitudes.forEachIndexed { index, amp ->
            val barHeight = amp * size.height
            drawLine(
                color = Color(0xFF4CAF50),
                start = Offset(index * barWidth, (size.height - barHeight) / 2),
                end = Offset(index * barWidth, (size.height + barHeight) / 2),
                strokeWidth = barWidth * 0.8f,
                cap = StrokeCap.Round
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `MediaPlayer` | 音声再生 |
| `MediaRecorder` | 録音 |
| `AudioFocusRequest` | オーディオフォーカス |
| `Canvas` | 波形描画 |

- `MediaPlayer`でローカル/リモート音声を再生
- `MediaRecorder`でAAC録音
- AudioFocusで他アプリとの共存制御
- Canvas Composableで波形をリアルタイム描画

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Media3](https://zenn.dev/myougatheaxo/articles/android-compose-compose-media3-2026)
- [Compose ExoPlayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-exoplayer-2026)
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
