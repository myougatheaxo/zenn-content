---
title: "QRスキャナー完全ガイド — CameraX/ML Kit/QR生成/Compose統合"
emoji: "📷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camera"]
published: true
---

## この記事で学べること

**QRスキャナー**（CameraX + ML Kit Barcode、QR生成、スキャン結果処理、Compose統合）を解説します。

---

## CameraX + ML Kit スキャナー

```kotlin
@Composable
fun QrScannerScreen(onScanned: (String) -> Unit) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var scannedValue by remember { mutableStateOf<String?>(null) }

    AndroidView(
        factory = { ctx ->
            PreviewView(ctx).apply {
                val cameraProviderFuture = ProcessCameraProvider.getInstance(ctx)
                cameraProviderFuture.addListener({
                    val cameraProvider = cameraProviderFuture.get()

                    val preview = Preview.Builder().build().also {
                        it.surfaceProvider = surfaceProvider
                    }

                    val barcodeScanner = BarcodeScanning.getClient(
                        BarcodeScannerOptions.Builder()
                            .setBarcodeFormats(Barcode.FORMAT_QR_CODE)
                            .build()
                    )

                    val imageAnalysis = ImageAnalysis.Builder()
                        .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                        .build()
                        .also { analysis ->
                            analysis.setAnalyzer(Executors.newSingleThreadExecutor()) { imageProxy ->
                                val mediaImage = imageProxy.image ?: run { imageProxy.close(); return@setAnalyzer }
                                val inputImage = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)

                                barcodeScanner.process(inputImage)
                                    .addOnSuccessListener { barcodes ->
                                        barcodes.firstOrNull()?.rawValue?.let { value ->
                                            if (scannedValue == null) {
                                                scannedValue = value
                                                onScanned(value)
                                            }
                                        }
                                    }
                                    .addOnCompleteListener { imageProxy.close() }
                            }
                        }

                    cameraProvider.unbindAll()
                    cameraProvider.bindToLifecycle(
                        lifecycleOwner, CameraSelector.DEFAULT_BACK_CAMERA,
                        preview, imageAnalysis
                    )
                }, ContextCompat.getMainExecutor(ctx))
            }
        },
        modifier = Modifier.fillMaxSize()
    )

    // スキャンオーバーレイ
    ScanOverlay()
}
```

---

## スキャンオーバーレイ

```kotlin
@Composable
fun ScanOverlay() {
    Canvas(Modifier.fillMaxSize()) {
        val scanAreaSize = 250.dp.toPx()
        val left = (size.width - scanAreaSize) / 2
        val top = (size.height - scanAreaSize) / 2

        // 半透明背景
        drawRect(Color.Black.copy(alpha = 0.5f))

        // スキャンエリア（透明）
        drawRect(
            Color.Transparent,
            topLeft = Offset(left, top),
            size = Size(scanAreaSize, scanAreaSize),
            blendMode = BlendMode.Clear
        )

        // 角のマーカー
        val cornerLength = 30.dp.toPx()
        val strokeWidth = 4.dp.toPx()
        val cornerColor = Color.White

        // 左上
        drawLine(cornerColor, Offset(left, top), Offset(left + cornerLength, top), strokeWidth)
        drawLine(cornerColor, Offset(left, top), Offset(left, top + cornerLength), strokeWidth)
        // 右上・左下・右下も同様
    }
}
```

---

## QR生成

```kotlin
@Composable
fun QrCodeImage(content: String, size: Dp = 200.dp) {
    val bitmap = remember(content) {
        val writer = QRCodeWriter()
        val bitMatrix = writer.encode(content, BarcodeFormat.QR_CODE, 512, 512)
        val bmp = Bitmap.createBitmap(512, 512, Bitmap.Config.RGB_565)
        for (x in 0 until 512) {
            for (y in 0 until 512) {
                bmp.setPixel(x, y, if (bitMatrix[x, y]) android.graphics.Color.BLACK else android.graphics.Color.WHITE)
            }
        }
        bmp.asImageBitmap()
    }

    Image(
        bitmap = bitmap,
        contentDescription = "QRコード",
        modifier = Modifier.size(size).clip(RoundedCornerShape(8.dp))
    )
}
```

---

## まとめ

| 機能 | ライブラリ |
|------|-----------|
| スキャン | CameraX + ML Kit |
| QR生成 | ZXing |
| 権限 | Camera Permission |
| オーバーレイ | Canvas |

- CameraX + ML Kit BarcodeでリアルタイムQRスキャン
- ZXingでQRコード画像を生成
- Canvasでスキャン領域オーバーレイ
- `ImageAnalysis`で効率的なフレーム処理

---

8種類のAndroidアプリテンプレート（カメラ機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camera-compose-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-custom-drawing-2026)
