---
title: "ML Kit顔検出完全ガイド — FaceDetector/ランドマーク/表情分類/リアルタイム"
emoji: "😀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "mlkit"]
published: true
---

## この記事で学べること

**ML Kit顔検出**（FaceDetector設定、ランドマーク取得、表情分類、CameraXリアルタイム検出）を解説します。

---

## FaceDetector設定

```kotlin
class FaceDetectionHelper @Inject constructor() {
    private val detector: FaceDetector

    init {
        val options = FaceDetectorOptions.Builder()
            .setPerformanceMode(FaceDetectorOptions.PERFORMANCE_MODE_FAST)
            .setLandmarkMode(FaceDetectorOptions.LANDMARK_MODE_ALL)
            .setClassificationMode(FaceDetectorOptions.CLASSIFICATION_MODE_ALL)
            .setMinFaceSize(0.15f)
            .enableTracking()
            .build()

        detector = FaceDetection.getClient(options)
    }

    suspend fun detectFaces(image: InputImage): List<FaceResult> {
        val faces = detector.process(image).await()
        return faces.map { face ->
            FaceResult(
                boundingBox = face.boundingBox,
                smilingProbability = face.smilingProbability,
                leftEyeOpenProbability = face.leftEyeOpenProbability,
                rightEyeOpenProbability = face.rightEyeOpenProbability,
                headEulerAngleY = face.headEulerAngleY,
                trackingId = face.trackingId
            )
        }
    }
}

data class FaceResult(
    val boundingBox: Rect,
    val smilingProbability: Float?,
    val leftEyeOpenProbability: Float?,
    val rightEyeOpenProbability: Float?,
    val headEulerAngleY: Float,
    val trackingId: Int?
)
```

---

## CameraXリアルタイム検出

```kotlin
class FaceAnalyzer(
    private val onFacesDetected: (List<FaceResult>) -> Unit
) : ImageAnalysis.Analyzer {
    private val helper = FaceDetectionHelper()

    @OptIn(ExperimentalGetImage::class)
    override fun analyze(imageProxy: ImageProxy) {
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()
            return
        }

        val image = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)

        CoroutineScope(Dispatchers.Default).launch {
            try {
                val faces = helper.detectFaces(image)
                onFacesDetected(faces)
            } finally {
                imageProxy.close()
            }
        }
    }
}
```

---

## 検出結果表示

```kotlin
@Composable
fun FaceDetectionOverlay(faces: List<FaceResult>, imageSize: IntSize) {
    Canvas(Modifier.fillMaxSize()) {
        faces.forEach { face ->
            val scaleX = size.width / imageSize.width
            val scaleY = size.height / imageSize.height
            val rect = face.boundingBox

            drawRect(
                color = Color.Green,
                topLeft = Offset(rect.left * scaleX, rect.top * scaleY),
                size = Size(rect.width() * scaleX, rect.height() * scaleY),
                style = Stroke(width = 3f)
            )

            face.smilingProbability?.let { smile ->
                drawContext.canvas.nativeCanvas.drawText(
                    "😊 ${(smile * 100).toInt()}%",
                    rect.left * scaleX,
                    rect.top * scaleY - 10,
                    android.graphics.Paint().apply {
                        color = android.graphics.Color.GREEN
                        textSize = 32f
                    }
                )
            }
        }
    }
}
```

---

## まとめ

| 機能 | 設定 |
|------|------|
| ランドマーク | `LANDMARK_MODE_ALL` |
| 表情分類 | `CLASSIFICATION_MODE_ALL` |
| トラッキング | `enableTracking()` |
| 最小サイズ | `setMinFaceSize(0.15f)` |

- ML Kit FaceDetectorでオフライン顔検出
- ランドマーク（目、鼻、口）の座標取得
- 笑顔・目の開閉を分類
- CameraX ImageAnalysisでリアルタイム処理

---

8種類のAndroidアプリテンプレート（ML Kit対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ML Kitテキスト認識](https://zenn.dev/myougatheaxo/articles/android-compose-mlkit-text-recognition-2026)
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camera-compose-2026)
- [QRスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-qr-scanner-2026)
