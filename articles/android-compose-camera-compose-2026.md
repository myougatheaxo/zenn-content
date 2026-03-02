---
title: "CameraX完全ガイド — プレビュー/撮影/動画録画/Compose統合"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camera"]
published: true
---

## この記事で学べること

**CameraX**（プレビュー表示、写真撮影、動画録画、フロント/バック切替、Compose統合）を解説します。

---

## カメラプレビュー

```kotlin
@Composable
fun CameraPreview(
    modifier: Modifier = Modifier,
    cameraSelector: CameraSelector = CameraSelector.DEFAULT_BACK_CAMERA,
    onImageCaptureReady: (ImageCapture) -> Unit = {}
) {
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

                    val imageCapture = ImageCapture.Builder()
                        .setCaptureMode(ImageCapture.CAPTURE_MODE_MAXIMIZE_QUALITY)
                        .build()

                    onImageCaptureReady(imageCapture)

                    cameraProvider.unbindAll()
                    cameraProvider.bindToLifecycle(
                        lifecycleOwner, cameraSelector, preview, imageCapture
                    )
                }, ContextCompat.getMainExecutor(ctx))
            }
        },
        modifier = modifier
    )
}
```

---

## 写真撮影

```kotlin
@Composable
fun CameraScreen() {
    var imageCapture by remember { mutableStateOf<ImageCapture?>(null) }
    var capturedUri by remember { mutableStateOf<Uri?>(null) }
    val context = LocalContext.current
    val scope = rememberCoroutineScope()

    Box(Modifier.fillMaxSize()) {
        CameraPreview(
            modifier = Modifier.fillMaxSize(),
            onImageCaptureReady = { imageCapture = it }
        )

        // 撮影ボタン
        IconButton(
            onClick = {
                val capture = imageCapture ?: return@IconButton
                val outputOptions = ImageCapture.OutputFileOptions.Builder(
                    context.contentResolver,
                    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
                    ContentValues().apply {
                        put(MediaStore.Images.Media.DISPLAY_NAME, "IMG_${System.currentTimeMillis()}.jpg")
                        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
                    }
                ).build()

                capture.takePicture(outputOptions, ContextCompat.getMainExecutor(context),
                    object : ImageCapture.OnImageSavedCallback {
                        override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                            capturedUri = output.savedUri
                        }
                        override fun onError(exception: ImageCaptureException) { /* エラー処理 */ }
                    }
                )
            },
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .padding(32.dp)
                .size(72.dp)
                .background(Color.White, CircleShape)
        ) {
            Box(
                Modifier.size(60.dp).background(Color.White, CircleShape)
                    .border(3.dp, Color.Gray, CircleShape)
            )
        }
    }
}
```

---

## カメラ切替

```kotlin
@Composable
fun SwitchableCameraScreen() {
    var isBackCamera by remember { mutableStateOf(true) }
    val cameraSelector = if (isBackCamera)
        CameraSelector.DEFAULT_BACK_CAMERA
    else
        CameraSelector.DEFAULT_FRONT_CAMERA

    Box(Modifier.fillMaxSize()) {
        CameraPreview(cameraSelector = cameraSelector)

        IconButton(
            onClick = { isBackCamera = !isBackCamera },
            modifier = Modifier.align(Alignment.TopEnd).padding(16.dp)
        ) {
            Icon(Icons.Default.Cameraswitch, "カメラ切替", tint = Color.White)
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| プレビュー | `Preview` + `PreviewView` |
| 撮影 | `ImageCapture` |
| 録画 | `VideoCapture` |
| 解析 | `ImageAnalysis` |

- `AndroidView`でCameraXプレビューをCompose内に配置
- `ImageCapture`でMediaStoreに保存
- `CameraSelector`でフロント/バック切替
- 権限は`rememberPermissionState`で管理

---

8種類のAndroidアプリテンプレート（カメラ機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [QRスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-qr-scanner-2026)
- [Photo Picker](https://zenn.dev/myougatheaxo/articles/android-compose-photo-picker-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
