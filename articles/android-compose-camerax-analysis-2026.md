---
title: "CameraX画像解析完全ガイド — ImageAnalysis/バーコード/物体検出/リアルタイム"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camerax"]
published: true
---

## この記事で学べること

**CameraX画像解析**（ImageAnalysis、カスタムAnalyzer、ML Kit連携、リアルタイム処理）を解説します。

---

## ImageAnalysis基本

```kotlin
@Composable
fun AnalysisCameraPreview(analyzer: ImageAnalysis.Analyzer) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    AndroidView(
        factory = { ctx ->
            PreviewView(ctx).also { previewView ->
                val cameraProviderFuture = ProcessCameraProvider.getInstance(ctx)
                cameraProviderFuture.addListener({
                    val cameraProvider = cameraProviderFuture.get()

                    val preview = Preview.Builder().build().also {
                        it.surfaceProvider = previewView.surfaceProvider
                    }

                    val imageAnalysis = ImageAnalysis.Builder()
                        .setTargetResolution(Size(1280, 720))
                        .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                        .build()
                        .also { it.setAnalyzer(Executors.newSingleThreadExecutor(), analyzer) }

                    cameraProvider.unbindAll()
                    cameraProvider.bindToLifecycle(
                        lifecycleOwner,
                        CameraSelector.DEFAULT_BACK_CAMERA,
                        preview,
                        imageAnalysis
                    )
                }, ContextCompat.getMainExecutor(ctx))
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

---

## カスタムAnalyzer

```kotlin
class BrightnessAnalyzer(
    private val onResult: (Double) -> Unit
) : ImageAnalysis.Analyzer {

    @OptIn(ExperimentalGetImage::class)
    override fun analyze(imageProxy: ImageProxy) {
        val buffer = imageProxy.planes[0].buffer
        val data = ByteArray(buffer.remaining())
        buffer.get(data)

        val brightness = data.map { it.toInt() and 0xFF }.average()
        onResult(brightness)

        imageProxy.close()
    }
}

// 色検出Analyzer
class DominantColorAnalyzer(
    private val onColorDetected: (Color) -> Unit
) : ImageAnalysis.Analyzer {

    @OptIn(ExperimentalGetImage::class)
    override fun analyze(imageProxy: ImageProxy) {
        val bitmap = imageProxy.toBitmap()

        val palette = Palette.from(bitmap).generate()
        palette.dominantSwatch?.let { swatch ->
            onColorDetected(Color(swatch.rgb))
        }

        imageProxy.close()
    }
}
```

---

## ML Kit物体検出

```kotlin
class ObjectDetectionAnalyzer(
    private val onObjectsDetected: (List<DetectedObject>) -> Unit
) : ImageAnalysis.Analyzer {
    private val detector = ObjectDetection.getClient(
        ObjectDetectorOptions.Builder()
            .setDetectorMode(ObjectDetectorOptions.STREAM_MODE)
            .enableMultipleObjects()
            .enableClassification()
            .build()
    )

    @OptIn(ExperimentalGetImage::class)
    override fun analyze(imageProxy: ImageProxy) {
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()
            return
        }

        val image = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)

        detector.process(image)
            .addOnSuccessListener { objects -> onObjectsDetected(objects) }
            .addOnCompleteListener { imageProxy.close() }
    }
}
```

---

## まとめ

| 解析タイプ | 用途 |
|-----------|------|
| 明るさ | 露出制御 |
| 色検出 | UIテーマ連動 |
| バーコード | 商品スキャン |
| 物体検出 | AR/分類 |

- `ImageAnalysis.Analyzer`でフレーム単位処理
- `STRATEGY_KEEP_ONLY_LATEST`でバックプレッシャー制御
- ML Kitと組み合わせてリアルタイムAI解析
- カスタムAnalyzerで独自の画像処理

---

8種類のAndroidアプリテンプレート（カメラ機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX Compose](https://zenn.dev/myougatheaxo/articles/android-compose-camera-compose-2026)
- [QRスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-qr-scanner-2026)
- [ML Kit顔検出](https://zenn.dev/myougatheaxo/articles/android-compose-mlkit-face-detection-2026)
