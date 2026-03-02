---
title: "QRコード読み取り/生成ガイド — CameraX + ML Kit"
emoji: "📷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camera"]
published: true
---

## この記事で学べること

CameraX + ML Kitでの**QRコード読み取り**とZXingでの**QRコード生成**を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.camera:camera-camera2:1.4.1")
    implementation("androidx.camera:camera-lifecycle:1.4.1")
    implementation("androidx.camera:camera-view:1.4.1")
    implementation("com.google.mlkit:barcode-scanning:17.3.0")
    implementation("com.google.zxing:core:3.5.3") // QR生成用
}
```

---

## QRコードスキャナー

```kotlin
@Composable
fun QrCodeScanner(onScanned: (String) -> Unit) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var scannedValue by remember { mutableStateOf<String?>(null) }

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (!granted) { /* パーミッション拒否処理 */ }
    }

    LaunchedEffect(Unit) {
        permissionLauncher.launch(Manifest.permission.CAMERA)
    }

    AndroidView(
        factory = { ctx ->
            val previewView = PreviewView(ctx)
            val cameraProviderFuture = ProcessCameraProvider.getInstance(ctx)

            cameraProviderFuture.addListener({
                val cameraProvider = cameraProviderFuture.get()
                val preview = Preview.Builder().build().also {
                    it.surfaceProvider = previewView.surfaceProvider
                }

                val analyzer = ImageAnalysis.Builder()
                    .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                    .build().also {
                        it.setAnalyzer(Executors.newSingleThreadExecutor()) { imageProxy ->
                            processImage(imageProxy) { value ->
                                if (scannedValue == null) {
                                    scannedValue = value
                                    onScanned(value)
                                }
                            }
                        }
                    }

                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    lifecycleOwner, CameraSelector.DEFAULT_BACK_CAMERA,
                    preview, analyzer
                )
            }, ContextCompat.getMainExecutor(ctx))

            previewView
        },
        modifier = Modifier.fillMaxSize()
    )
}

@OptIn(ExperimentalGetImage::class)
private fun processImage(imageProxy: ImageProxy, onResult: (String) -> Unit) {
    val mediaImage = imageProxy.image ?: run { imageProxy.close(); return }
    val inputImage = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)
    val scanner = BarcodeScanning.getClient()

    scanner.process(inputImage)
        .addOnSuccessListener { barcodes ->
            barcodes.firstOrNull()?.rawValue?.let { onResult(it) }
        }
        .addOnCompleteListener { imageProxy.close() }
}
```

---

## QRコード生成

```kotlin
@Composable
fun QrCodeGenerator(content: String, size: Int = 256) {
    val bitmap = remember(content) {
        generateQrCode(content, size)
    }

    bitmap?.let {
        Image(
            bitmap = it.asImageBitmap(),
            contentDescription = "QRコード",
            modifier = Modifier.size(size.dp)
        )
    }
}

fun generateQrCode(content: String, size: Int): Bitmap? {
    return try {
        val hints = mapOf(
            EncodeHintType.CHARACTER_SET to "UTF-8",
            EncodeHintType.MARGIN to 1
        )
        val bitMatrix = MultiFormatWriter().encode(
            content, BarcodeFormat.QR_CODE, size, size, hints
        )
        val bitmap = Bitmap.createBitmap(size, size, Bitmap.Config.RGB_565)
        for (x in 0 until size) {
            for (y in 0 until size) {
                bitmap.setPixel(x, y, if (bitMatrix[x, y]) Color.BLACK else Color.WHITE)
            }
        }
        bitmap
    } catch (e: Exception) {
        null
    }
}
```

---

## スキャン結果画面

```kotlin
@Composable
fun ScanResultScreen() {
    var scannedResult by remember { mutableStateOf<String?>(null) }
    var showScanner by remember { mutableStateOf(true) }

    if (showScanner && scannedResult == null) {
        QrCodeScanner(
            onScanned = { value ->
                scannedResult = value
                showScanner = false
            }
        )
    } else {
        Column(
            Modifier.fillMaxSize().padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Icon(Icons.Default.QrCodeScanner, null, Modifier.size(64.dp))
            Spacer(Modifier.height(16.dp))
            Text("スキャン結果", style = MaterialTheme.typography.titleLarge)
            Spacer(Modifier.height(8.dp))

            SelectionContainer {
                Text(scannedResult ?: "", style = MaterialTheme.typography.bodyLarge)
            }

            Spacer(Modifier.height(24.dp))
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                OutlinedButton(onClick = {
                    scannedResult = null
                    showScanner = true
                }) { Text("再スキャン") }

                Button(onClick = {
                    // URLならブラウザで開く
                    scannedResult?.let { /* 処理 */ }
                }) { Text("開く") }
            }
        }
    }
}
```

---

## まとめ

- CameraX + ML Kit `BarcodeScanning`でQR読み取り
- `ImageAnalysis`で画像フレームをリアルタイム分析
- ZXing `MultiFormatWriter`でQRコード生成
- `STRATEGY_KEEP_ONLY_LATEST`で最新フレームのみ処理
- カメラパーミッション管理は`rememberLauncherForActivityResult`
- スキャン結果はSelectionContainerでコピー可能に

---

8種類のAndroidアプリテンプレート（カメラ機能対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX Composeガイド](https://zenn.dev/myougatheaxo/articles/android-camerax-compose-2026)
- [パーミッション管理ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handler-2026)
- [画像ピッカーガイド](https://zenn.dev/myougatheaxo/articles/android-compose-image-picker-2026)
