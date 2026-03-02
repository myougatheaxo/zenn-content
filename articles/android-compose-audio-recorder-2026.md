---
title: "音声録音完全ガイド — MediaRecorder/AudioRecord/波形表示/Compose"
emoji: "🎙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "audio"]
published: true
---

## この記事で学べること

**音声録音**（MediaRecorder、AudioRecord、波形表示、録音時間表示、ファイル保存）を解説します。

---

## MediaRecorder

```kotlin
class AudioRecorder @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private var recorder: MediaRecorder? = null
    private var outputFile: File? = null

    fun start(): File {
        val file = File(context.cacheDir, "recording_${System.currentTimeMillis()}.m4a")
        outputFile = file

        recorder = MediaRecorder(context).apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setAudioSamplingRate(44100)
            setAudioEncodingBitRate(128000)
            setOutputFile(file.absolutePath)
            prepare()
            start()
        }

        return file
    }

    fun stop(): File? {
        recorder?.apply {
            stop()
            release()
        }
        recorder = null
        return outputFile
    }

    fun getAmplitude(): Int = recorder?.maxAmplitude ?: 0
}
```

---

## 録音画面

```kotlin
@Composable
fun RecorderScreen(viewModel: RecorderViewModel = hiltViewModel()) {
    val isRecording by viewModel.isRecording.collectAsStateWithLifecycle()
    val duration by viewModel.duration.collectAsStateWithLifecycle()
    val amplitudes by viewModel.amplitudes.collectAsStateWithLifecycle()

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        // 波形表示
        WaveformView(amplitudes, Modifier.fillMaxWidth().height(100.dp))

        Spacer(Modifier.height(24.dp))

        // 録音時間
        Text(
            formatDuration(duration),
            style = MaterialTheme.typography.displaySmall,
            color = if (isRecording) MaterialTheme.colorScheme.error else MaterialTheme.colorScheme.onSurface
        )

        Spacer(Modifier.height(32.dp))

        // 録音ボタン
        FloatingActionButton(
            onClick = { if (isRecording) viewModel.stop() else viewModel.start() },
            containerColor = if (isRecording) MaterialTheme.colorScheme.error else MaterialTheme.colorScheme.primary,
            modifier = Modifier.size(72.dp)
        ) {
            Icon(
                if (isRecording) Icons.Default.Stop else Icons.Default.Mic,
                "録音",
                Modifier.size(32.dp)
            )
        }
    }
}

fun formatDuration(seconds: Long): String {
    val m = seconds / 60
    val s = seconds % 60
    return "%02d:%02d".format(m, s)
}
```

---

## 波形表示

```kotlin
@Composable
fun WaveformView(
    amplitudes: List<Float>,
    modifier: Modifier = Modifier
) {
    Canvas(modifier) {
        val barWidth = 4.dp.toPx()
        val gap = 2.dp.toPx()
        val maxBars = (size.width / (barWidth + gap)).toInt()
        val displayAmplitudes = amplitudes.takeLast(maxBars)

        displayAmplitudes.forEachIndexed { index, amplitude ->
            val barHeight = (amplitude * size.height).coerceIn(4.dp.toPx(), size.height)
            val x = index * (barWidth + gap)
            val y = (size.height - barHeight) / 2

            drawRoundRect(
                color = Color(0xFF4CAF50),
                topLeft = Offset(x, y),
                size = Size(barWidth, barHeight),
                cornerRadius = CornerRadius(2.dp.toPx())
            )
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 録音 | `MediaRecorder` |
| 振幅取得 | `maxAmplitude` |
| 波形表示 | `Canvas` |
| ファイル保存 | `cacheDir` / `MediaStore` |

- `MediaRecorder`でAAC/M4A形式の高品質録音
- `maxAmplitude`で振幅を取得し波形表示
- `Canvas`でリアルタイム波形描画
- 録音権限(`RECORD_AUDIO`)は必須

---

8種類のAndroidアプリテンプレート（オーディオ機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Media3](https://zenn.dev/myougatheaxo/articles/android-compose-media3-compose-2026)
- [音声認識](https://zenn.dev/myougatheaxo/articles/android-compose-speech-recognition-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
