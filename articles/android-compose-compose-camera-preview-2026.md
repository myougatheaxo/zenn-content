---
title: "CameraX Preview完全ガイド — カメラプレビュー/撮影/QRスキャン"
emoji: "📷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camerax"]
published: true
---

## この記事で学べること

**CameraX Preview**（カメラプレビュー、写真撮影、画像分析）を解説します。

---

## カメラプレビュー

```kotlin
@Composable
fun CameraPreview() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    val previewView = remember { PreviewView(context) }

    LaunchedEffect(Unit) {
        val cameraProvider = ProcessCameraProvider.awaitInstance(context)
        val preview = Preview.Builder().build().apply {
            surfaceProvider = previewView.surfaceProvider
        }
        cameraProvider.unbindAll()
        cameraProvider.bindToLifecycle(
            lifecycleOwner,
            CameraSelector.DEFAULT_BACK_CAMERA,
            preview
        )
    }

    AndroidView(factory = { previewView }, modifier = Modifier.fillMaxSize())
}
```

---

## 写真撮影

```kotlin
@Composable
fun CameraWithCapture() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var imageCapture by remember { mutableStateOf<ImageCapture?>(null) }
    var capturedUri by remember { mutableStateOf<Uri?>(null) }
    val previewView = remember { PreviewView(context) }

    LaunchedEffect(Unit) {
        val cameraProvider = ProcessCameraProvider.awaitInstance(context)
        val preview = Preview.Builder().build().apply {
            surfaceProvider = previewView.surfaceProvider
        }
        val capture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .build()
        imageCapture = capture
        cameraProvider.unbindAll()
        cameraProvider.bindToLifecycle(
            lifecycleOwner, CameraSelector.DEFAULT_BACK_CAMERA, preview, capture
        )
    }

    Box(Modifier.fillMaxSize()) {
        AndroidView(factory = { previewView }, Modifier.fillMaxSize())

        FloatingActionButton(
            onClick = {
                val file = File(context.cacheDir, "photo_${System.currentTimeMillis()}.jpg")
                val outputOptions = ImageCapture.OutputFileOptions.Builder(file).build()
                imageCapture?.takePicture(
                    outputOptions, ContextCompat.getMainExecutor(context),
                    object : ImageCapture.OnImageSavedCallback {
                        override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                            capturedUri = Uri.fromFile(file)
                        }
                        override fun onError(exception: ImageCaptureException) {}
                    }
                )
            },
            modifier = Modifier.align(Alignment.BottomCenter).padding(32.dp)
        ) {
            Icon(Icons.Default.Camera, "撮影")
        }
    }
}
```

---

## 画像分析（QR）

```kotlin
@Composable
fun QrScannerCamera(onQrDetected: (String) -> Unit) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    val previewView = remember { PreviewView(context) }

    LaunchedEffect(Unit) {
        val cameraProvider = ProcessCameraProvider.awaitInstance(context)
        val preview = Preview.Builder().build().apply {
            surfaceProvider = previewView.surfaceProvider
        }
        val analyzer = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build().apply {
                setAnalyzer(ContextCompat.getMainExecutor(context)) { imageProxy ->
                    val scanner = BarcodeScanning.getClient()
                    val inputImage = InputImage.fromMediaImage(
                        imageProxy.image!!, imageProxy.imageInfo.rotationDegrees
                    )
                    scanner.process(inputImage)
                        .addOnSuccessListener { barcodes ->
                            barcodes.firstOrNull()?.rawValue?.let { onQrDetected(it) }
                        }
                        .addOnCompleteListener { imageProxy.close() }
                }
            }
        cameraProvider.unbindAll()
        cameraProvider.bindToLifecycle(lifecycleOwner, CameraSelector.DEFAULT_BACK_CAMERA, preview, analyzer)
    }

    AndroidView(factory = { previewView }, Modifier.fillMaxSize())
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `PreviewView` | カメラプレビュー |
| `ImageCapture` | 写真撮影 |
| `ImageAnalysis` | 画像分析 |
| `CameraSelector` | カメラ選択 |

- CameraXでカメラプレビューをCompose内に表示
- `ImageCapture`で写真撮影
- `ImageAnalysis`でQRコードスキャン等
- ライフサイクル自動管理で安全

---

8種類のAndroidアプリテンプレート（カメラ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [パーミッション管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [Video Player](https://zenn.dev/myougatheaxo/articles/android-compose-compose-video-player-2026)
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
