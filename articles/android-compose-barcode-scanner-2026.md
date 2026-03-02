---
title: "バーコード/QRコードスキャナー — ML Kit + CameraX + Compose"
emoji: "📷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "mlkit"]
published: true
---

## この記事で学べること

**ML Kit**のバーコードスキャン（QRコード、CameraX統合、結果処理、Compose UI）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.mlkit:barcode-scanning:17.3.0")
    implementation("androidx.camera:camera-core:1.4.0")
    implementation("androidx.camera:camera-camera2:1.4.0")
    implementation("androidx.camera:camera-lifecycle:1.4.0")
    implementation("androidx.camera:camera-view:1.4.0")
    implementation("androidx.camera:camera-mlkit-vision:1.4.0")
}
```

---

## バーコードスキャナー

```kotlin
@Composable
fun BarcodeScannerScreen(
    onBarcodeDetected: (String) -> Unit
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var detectedBarcode by remember { mutableStateOf<String?>(null) }

    val cameraPermission = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        // パーミッション結果
    }

    LaunchedEffect(Unit) {
        cameraPermission.launch(Manifest.permission.CAMERA)
    }

    Box(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { ctx ->
                PreviewView(ctx).also { previewView ->
                    val cameraProviderFuture = ProcessCameraProvider.getInstance(ctx)

                    cameraProviderFuture.addListener({
                        val cameraProvider = cameraProviderFuture.get()

                        val preview = Preview.Builder().build().also {
                            it.setSurfaceProvider(previewView.surfaceProvider)
                        }

                        val barcodeScanner = BarcodeScanning.getClient(
                            BarcodeScannerOptions.Builder()
                                .setBarcodeFormats(
                                    Barcode.FORMAT_QR_CODE,
                                    Barcode.FORMAT_EAN_13,
                                    Barcode.FORMAT_EAN_8
                                )
                                .build()
                        )

                        val analysis = ImageAnalysis.Builder()
                            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                            .build()
                            .also { imageAnalysis ->
                                imageAnalysis.setAnalyzer(
                                    ContextCompat.getMainExecutor(ctx)
                                ) { imageProxy ->
                                    processImage(imageProxy, barcodeScanner) { barcode ->
                                        if (detectedBarcode == null) {
                                            detectedBarcode = barcode
                                            onBarcodeDetected(barcode)
                                        }
                                    }
                                }
                            }

                        cameraProvider.unbindAll()
                        cameraProvider.bindToLifecycle(
                            lifecycleOwner,
                            CameraSelector.DEFAULT_BACK_CAMERA,
                            preview,
                            analysis
                        )
                    }, ContextCompat.getMainExecutor(ctx))
                }
            },
            modifier = Modifier.fillMaxSize()
        )

        // スキャンガイド枠
        Box(
            Modifier.fillMaxSize(),
            contentAlignment = Alignment.Center
        ) {
            Box(
                Modifier
                    .size(250.dp)
                    .border(2.dp, Color.White, RoundedCornerShape(16.dp))
            )
        }

        // 検出結果
        detectedBarcode?.let { barcode ->
            Card(
                Modifier.align(Alignment.BottomCenter).padding(16.dp).fillMaxWidth()
            ) {
                Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.QrCode, null)
                    Spacer(Modifier.width(8.dp))
                    Text(barcode, style = MaterialTheme.typography.bodyLarge)
                }
            }
        }
    }
}

@OptIn(ExperimentalGetImage::class)
private fun processImage(
    imageProxy: ImageProxy,
    scanner: com.google.mlkit.vision.barcode.BarcodeScanner,
    onDetected: (String) -> Unit
) {
    val mediaImage = imageProxy.image ?: run {
        imageProxy.close()
        return
    }

    val inputImage = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)

    scanner.process(inputImage)
        .addOnSuccessListener { barcodes ->
            barcodes.firstOrNull()?.rawValue?.let { onDetected(it) }
        }
        .addOnCompleteListener {
            imageProxy.close()
        }
}
```

---

## QRコード生成

```kotlin
@Composable
fun QrCodeGenerator(content: String, size: Int = 256) {
    val bitmap = remember(content) {
        val writer = MultiFormatWriter()
        val bitMatrix = writer.encode(content, BarcodeFormat.QR_CODE, size, size)
        val bmp = Bitmap.createBitmap(size, size, Bitmap.Config.RGB_565)
        for (x in 0 until size) {
            for (y in 0 until size) {
                bmp.setPixel(x, y, if (bitMatrix[x, y]) Color.BLACK else Color.WHITE)
            }
        }
        bmp.asImageBitmap()
    }

    Image(
        bitmap = bitmap,
        contentDescription = "QRコード",
        modifier = Modifier.size(size.dp)
    )
}

// 使用
QrCodeGenerator(content = "https://myougatheax.gumroad.com")
```

---

## まとめ

- `ML Kit BarcodeScanning`でQR/EAN/Code128等対応
- `CameraX ImageAnalysis`でリアルタイムフレーム解析
- `STRATEGY_KEEP_ONLY_LATEST`でフレーム落ち防止
- `PreviewView`を`AndroidView`でCompose統合
- `MultiFormatWriter`でQRコード生成
- パーミッション`CAMERA`必須

---

8種類のAndroidアプリテンプレート（カメラ機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [QRコードスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-qrcode-scanner-2026)
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-camerax-compose-2026)
- [画像ピッカー](https://zenn.dev/myougatheaxo/articles/android-compose-image-picker-camera-2026)
