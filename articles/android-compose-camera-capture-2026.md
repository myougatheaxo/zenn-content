---
title: "CameraX + Compose完全ガイド — 撮影/プレビュー/分析"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camerax"]
published: true
---

## この記事で学べること

**CameraX + Compose**（カメラプレビュー、写真撮影、画像分析、QRコード読み取り）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.camera:camera-camera2:1.4.1")
    implementation("androidx.camera:camera-lifecycle:1.4.1")
    implementation("androidx.camera:camera-view:1.4.1")

    // パーミッション
    implementation("com.google.accompanist:accompanist-permissions:0.36.0")
}
```

---

## カメラプレビュー

```kotlin
@Composable
fun CameraPreviewScreen() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    val cameraPermissionState = rememberPermissionState(Manifest.permission.CAMERA)

    if (cameraPermissionState.status.isGranted) {
        CameraPreview(context, lifecycleOwner)
    } else {
        Column(
            Modifier.fillMaxSize(),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text("カメラの許可が必要です")
            Button(onClick = { cameraPermissionState.launchPermissionRequest() }) {
                Text("許可する")
            }
        }
    }
}

@Composable
fun CameraPreview(context: Context, lifecycleOwner: LifecycleOwner) {
    val previewView = remember { PreviewView(context) }

    LaunchedEffect(Unit) {
        val cameraProvider = context.getCameraProvider()

        val preview = Preview.Builder().build().also {
            it.surfaceProvider = previewView.surfaceProvider
        }

        val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

        cameraProvider.unbindAll()
        cameraProvider.bindToLifecycle(
            lifecycleOwner, cameraSelector, preview
        )
    }

    AndroidView(
        factory = { previewView },
        modifier = Modifier.fillMaxSize()
    )
}

suspend fun Context.getCameraProvider(): ProcessCameraProvider =
    suspendCancellableCoroutine { cont ->
        val future = ProcessCameraProvider.getInstance(this)
        future.addListener(
            { cont.resume(future.get()) },
            ContextCompat.getMainExecutor(this)
        )
    }
```

---

## 写真撮影

```kotlin
@Composable
fun CameraCaptureScreen() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var imageCapture by remember { mutableStateOf<ImageCapture?>(null) }
    var capturedUri by remember { mutableStateOf<Uri?>(null) }

    val previewView = remember { PreviewView(context) }

    LaunchedEffect(Unit) {
        val cameraProvider = context.getCameraProvider()

        val preview = Preview.Builder().build().also {
            it.surfaceProvider = previewView.surfaceProvider
        }

        val capture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MAXIMIZE_QUALITY)
            .build()

        imageCapture = capture

        cameraProvider.unbindAll()
        cameraProvider.bindToLifecycle(
            lifecycleOwner,
            CameraSelector.DEFAULT_BACK_CAMERA,
            preview, capture
        )
    }

    Box(Modifier.fillMaxSize()) {
        AndroidView(factory = { previewView }, modifier = Modifier.fillMaxSize())

        // 撮影ボタン
        FloatingActionButton(
            onClick = { takePhoto(context, imageCapture) { capturedUri = it } },
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .padding(32.dp)
        ) {
            Icon(Icons.Default.CameraAlt, contentDescription = "撮影")
        }
    }
}

private fun takePhoto(
    context: Context,
    imageCapture: ImageCapture?,
    onCaptured: (Uri) -> Unit
) {
    val capture = imageCapture ?: return

    val contentValues = ContentValues().apply {
        put(MediaStore.MediaColumns.DISPLAY_NAME, "IMG_${System.currentTimeMillis()}")
        put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
        put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/MyApp")
    }

    val outputOptions = ImageCapture.OutputFileOptions.Builder(
        context.contentResolver,
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        contentValues
    ).build()

    capture.takePicture(
        outputOptions,
        ContextCompat.getMainExecutor(context),
        object : ImageCapture.OnImageSavedCallback {
            override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                output.savedUri?.let(onCaptured)
            }
            override fun onError(exception: ImageCaptureException) {
                Log.e("Camera", "Capture failed", exception)
            }
        }
    )
}
```

---

## 画像分析（ImageAnalysis）

```kotlin
@Composable
fun CameraAnalysisScreen() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var brightness by remember { mutableFloatStateOf(0f) }

    val previewView = remember { PreviewView(context) }

    LaunchedEffect(Unit) {
        val cameraProvider = context.getCameraProvider()

        val preview = Preview.Builder().build().also {
            it.surfaceProvider = previewView.surfaceProvider
        }

        val analysis = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also { imageAnalysis ->
                imageAnalysis.setAnalyzer(
                    ContextCompat.getMainExecutor(context)
                ) { imageProxy ->
                    // 輝度計算
                    val buffer = imageProxy.planes[0].buffer
                    val bytes = ByteArray(buffer.remaining())
                    buffer.get(bytes)
                    val avg = bytes.map { it.toInt() and 0xFF }.average().toFloat()
                    brightness = avg
                    imageProxy.close()
                }
            }

        cameraProvider.unbindAll()
        cameraProvider.bindToLifecycle(
            lifecycleOwner,
            CameraSelector.DEFAULT_BACK_CAMERA,
            preview, analysis
        )
    }

    Box(Modifier.fillMaxSize()) {
        AndroidView(factory = { previewView }, modifier = Modifier.fillMaxSize())

        Text(
            text = "明るさ: ${brightness.toInt()}",
            modifier = Modifier.align(Alignment.TopCenter).padding(16.dp),
            color = Color.White,
            style = MaterialTheme.typography.headlineSmall
        )
    }
}
```

---

## まとめ

| 機能 | クラス |
|------|--------|
| プレビュー | `Preview` + `PreviewView` |
| 写真撮影 | `ImageCapture` |
| 画像分析 | `ImageAnalysis` |
| カメラ選択 | `CameraSelector` |
| ライフサイクル | `bindToLifecycle()` |

- `AndroidView`でPreviewViewをComposeに統合
- `ProcessCameraProvider`でカメラバインド
- `ImageCapture`でMediaStoreに保存
- `ImageAnalysis`でリアルタイムフレーム処理

---

8種類のAndroidアプリテンプレート（カメラ機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [バーコードスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-barcode-scanner-2026)
- [パーミッション管理](https://zenn.dev/myougatheaxo/articles/android-compose-share-intent-2026)
- [Compose⇔View相互運用](https://zenn.dev/myougatheaxo/articles/android-compose-compose-interop-2026)
