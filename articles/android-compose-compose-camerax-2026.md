---
title: "Compose CameraX完全ガイド — プレビュー/写真撮影/QRコード読取"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camerax"]
published: true
---

## この記事で学べること

**Compose CameraX**（カメラプレビュー、写真撮影、QRコード読取、画像分析）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("androidx.camera:camera-camera2:1.3.4")
    implementation("androidx.camera:camera-lifecycle:1.3.4")
    implementation("androidx.camera:camera-view:1.3.4")
}
```

---

## カメラプレビュー + 撮影

```kotlin
@Composable
fun CameraScreen() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var imageCapture by remember { mutableStateOf<ImageCapture?>(null) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (!granted) { /* パーミッション拒否時の処理 */ }
    }

    LaunchedEffect(Unit) {
        launcher.launch(Manifest.permission.CAMERA)
    }

    Box(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { ctx ->
                PreviewView(ctx).apply {
                    val cameraProviderFuture = ProcessCameraProvider.getInstance(ctx)
                    cameraProviderFuture.addListener({
                        val cameraProvider = cameraProviderFuture.get()
                        val preview = Preview.Builder().build().also {
                            it.surfaceProvider = surfaceProvider
                        }
                        imageCapture = ImageCapture.Builder()
                            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
                            .build()

                        cameraProvider.unbindAll()
                        cameraProvider.bindToLifecycle(
                            lifecycleOwner, CameraSelector.DEFAULT_BACK_CAMERA,
                            preview, imageCapture
                        )
                    }, ContextCompat.getMainExecutor(ctx))
                }
            },
            modifier = Modifier.fillMaxSize()
        )

        // 撮影ボタン
        FloatingActionButton(
            onClick = { takePhoto(context, imageCapture) },
            modifier = Modifier.align(Alignment.BottomCenter).padding(32.dp)
        ) { Icon(Icons.Default.CameraAlt, "撮影") }
    }
}

fun takePhoto(context: Context, imageCapture: ImageCapture?) {
    val outputOptions = ImageCapture.OutputFileOptions.Builder(
        File(context.cacheDir, "photo_${System.currentTimeMillis()}.jpg")
    ).build()
    imageCapture?.takePicture(outputOptions,
        ContextCompat.getMainExecutor(context),
        object : ImageCapture.OnImageSavedCallback {
            override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                // 保存成功
            }
            override fun onError(exc: ImageCaptureException) {
                // エラー処理
            }
        })
}
```

---

## QRコード読取

```kotlin
// implementation("com.google.mlkit:barcode-scanning:17.2.0")

@Composable
fun QrScannerScreen(onQrDetected: (String) -> Unit) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    AndroidView(
        factory = { ctx ->
            PreviewView(ctx).apply {
                val cameraProviderFuture = ProcessCameraProvider.getInstance(ctx)
                cameraProviderFuture.addListener({
                    val cameraProvider = cameraProviderFuture.get()
                    val preview = Preview.Builder().build().also {
                        it.surfaceProvider = surfaceProvider
                    }
                    val analyzer = ImageAnalysis.Builder()
                        .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                        .build().also {
                            it.setAnalyzer(Executors.newSingleThreadExecutor()) { imageProxy ->
                                processQr(imageProxy, onQrDetected)
                            }
                        }
                    cameraProvider.unbindAll()
                    cameraProvider.bindToLifecycle(
                        lifecycleOwner, CameraSelector.DEFAULT_BACK_CAMERA,
                        preview, analyzer
                    )
                }, ContextCompat.getMainExecutor(ctx))
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}

@OptIn(ExperimentalGetImage::class)
fun processQr(imageProxy: ImageProxy, onDetected: (String) -> Unit) {
    val mediaImage = imageProxy.image ?: run { imageProxy.close(); return }
    val inputImage = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)
    BarcodeScanning.getClient().process(inputImage)
        .addOnSuccessListener { barcodes ->
            barcodes.firstOrNull()?.rawValue?.let { onDetected(it) }
        }
        .addOnCompleteListener { imageProxy.close() }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ProcessCameraProvider` | カメラ管理 |
| `Preview` | プレビュー表示 |
| `ImageCapture` | 写真撮影 |
| `ImageAnalysis` | リアルタイム解析 |

- `AndroidView`+`PreviewView`でCompose内にカメラ表示
- `ImageCapture`で写真撮影
- `ImageAnalysis`+ML KitでQRコードリアルタイム読取
- `bindToLifecycle`でライフサイクル自動管理

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Permissions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permissions-2026)
- [Compose FilePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-file-picker-2026)
- [Compose ImageCropper](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-cropper-2026)
