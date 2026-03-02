---
title: "動画録画完全ガイド — CameraX VideoCapture/画面録画/Compose統合"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camera"]
published: true
---

## この記事で学べること

**動画録画**（CameraX VideoCapture、録画制御、画面録画、プログレス表示）を解説します。

---

## CameraX 動画録画

```kotlin
class VideoRecorder @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private var recording: Recording? = null

    @SuppressLint("MissingPermission")
    fun startRecording(
        videoCapture: VideoCapture<Recorder>,
        onEvent: (VideoRecordEvent) -> Unit
    ) {
        val outputOptions = MediaStoreOutputOptions.Builder(
            context.contentResolver,
            MediaStore.Video.Media.EXTERNAL_CONTENT_URI
        ).setContentValues(ContentValues().apply {
            put(MediaStore.Video.Media.DISPLAY_NAME, "VID_${System.currentTimeMillis()}")
            put(MediaStore.Video.Media.MIME_TYPE, "video/mp4")
        }).build()

        recording = videoCapture.output
            .prepareRecording(context, outputOptions)
            .withAudioEnabled()
            .start(ContextCompat.getMainExecutor(context)) { event ->
                onEvent(event)
            }
    }

    fun stopRecording() {
        recording?.stop()
        recording = null
    }

    fun pauseRecording() { recording?.pause() }
    fun resumeRecording() { recording?.resume() }
}
```

---

## 録画画面

```kotlin
@Composable
fun VideoRecordScreen() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var isRecording by remember { mutableStateOf(false) }
    var duration by remember { mutableLongStateOf(0L) }

    Box(Modifier.fillMaxSize()) {
        // カメラプレビュー
        AndroidView(
            factory = { ctx ->
                PreviewView(ctx).apply {
                    val cameraProviderFuture = ProcessCameraProvider.getInstance(ctx)
                    cameraProviderFuture.addListener({
                        val cameraProvider = cameraProviderFuture.get()
                        val preview = Preview.Builder().build().also {
                            it.surfaceProvider = surfaceProvider
                        }
                        val recorder = Recorder.Builder()
                            .setQualitySelector(QualitySelector.from(Quality.HD))
                            .build()
                        val videoCapture = VideoCapture.withOutput(recorder)

                        cameraProvider.unbindAll()
                        cameraProvider.bindToLifecycle(
                            lifecycleOwner, CameraSelector.DEFAULT_BACK_CAMERA,
                            preview, videoCapture
                        )
                    }, ContextCompat.getMainExecutor(ctx))
                }
            },
            modifier = Modifier.fillMaxSize()
        )

        // 録画時間表示
        if (isRecording) {
            Surface(
                color = Color.Black.copy(alpha = 0.5f),
                shape = RoundedCornerShape(16.dp),
                modifier = Modifier.align(Alignment.TopCenter).padding(16.dp)
            ) {
                Row(Modifier.padding(horizontal = 12.dp, vertical = 6.dp)) {
                    Box(
                        Modifier.size(8.dp).background(Color.Red, CircleShape)
                            .align(Alignment.CenterVertically)
                    )
                    Spacer(Modifier.width(8.dp))
                    Text(formatDuration(duration), color = Color.White)
                }
            }
        }

        // 録画ボタン
        Box(
            Modifier
                .align(Alignment.BottomCenter)
                .padding(32.dp)
                .size(72.dp)
                .clip(CircleShape)
                .background(if (isRecording) Color.Red else Color.White)
                .clickable { isRecording = !isRecording },
            contentAlignment = Alignment.Center
        ) {
            if (isRecording) {
                Box(Modifier.size(24.dp).background(Color.White, RoundedCornerShape(4.dp)))
            } else {
                Box(Modifier.size(56.dp).background(Color.Red, CircleShape))
            }
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 録画 | `VideoCapture<Recorder>` |
| 品質設定 | `QualitySelector` |
| 一時停止 | `Recording.pause/resume` |
| 保存先 | `MediaStoreOutputOptions` |

- CameraX `VideoCapture`で簡単に動画録画
- `QualitySelector`でHD/FHD/4K選択
- `pause/resume`で録画の一時停止
- `MediaStore`に保存してギャラリーに表示

---

8種類のAndroidアプリテンプレート（カメラ/動画対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camera-compose-2026)
- [Media3](https://zenn.dev/myougatheaxo/articles/android-compose-media3-compose-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
