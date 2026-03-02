---
title: "Compose ML Kit完全ガイド — テキスト認識/顔検出/バーコード/画像ラベリング"
emoji: "🤖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "mlkit"]
published: true
---

## この記事で学べること

**Compose ML Kit**（テキスト認識、顔検出、バーコードスキャン、画像ラベリング）を解説します。

---

## テキスト認識（OCR）

```groovy
// build.gradle
dependencies {
    implementation("com.google.mlkit:text-recognition-japanese:16.0.0")
}
```

```kotlin
@Composable
fun TextRecognitionScreen() {
    var recognizedText by remember { mutableStateOf("") }
    val context = LocalContext.current

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicturePreview()
    ) { bitmap ->
        bitmap?.let {
            val image = InputImage.fromBitmap(it, 0)
            val recognizer = TextRecognition.getClient(JapaneseTextRecognizerOptions.Builder().build())
            recognizer.process(image)
                .addOnSuccessListener { result -> recognizedText = result.text }
                .addOnFailureListener { e -> recognizedText = "エラー: ${e.message}" }
        }
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Button(onClick = { launcher.launch(null) }) { Text("写真を撮影してOCR") }
        if (recognizedText.isNotEmpty()) {
            ElevatedCard(Modifier.fillMaxWidth()) {
                Text(recognizedText, Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## 顔検出

```kotlin
// implementation("com.google.mlkit:face-detection:16.1.6")

@Composable
fun FaceDetectionScreen() {
    var faceCount by remember { mutableIntStateOf(0) }
    var smiling by remember { mutableStateOf(false) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicturePreview()
    ) { bitmap ->
        bitmap?.let {
            val image = InputImage.fromBitmap(it, 0)
            val options = FaceDetectorOptions.Builder()
                .setPerformanceMode(FaceDetectorOptions.PERFORMANCE_MODE_ACCURATE)
                .setClassificationMode(FaceDetectorOptions.CLASSIFICATION_MODE_ALL)
                .build()
            FaceDetection.getClient(options).process(image)
                .addOnSuccessListener { faces ->
                    faceCount = faces.size
                    smiling = faces.any { (it.smilingProbability ?: 0f) > 0.5f }
                }
        }
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Button(onClick = { launcher.launch(null) }) { Text("顔を検出") }
        Text("検出した顔: ${faceCount}人")
        if (smiling) Text("笑顔を検出しました！", color = Color(0xFF4CAF50))
    }
}
```

---

## 画像ラベリング

```kotlin
// implementation("com.google.mlkit:image-labeling:17.0.8")

@Composable
fun ImageLabelingScreen() {
    var labels by remember { mutableStateOf<List<Pair<String, Float>>>(emptyList()) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicturePreview()
    ) { bitmap ->
        bitmap?.let {
            val image = InputImage.fromBitmap(it, 0)
            ImageLabeling.getClient(ImageLabelerOptions.DEFAULT_OPTIONS).process(image)
                .addOnSuccessListener { imageLabels ->
                    labels = imageLabels.map { it.text to it.confidence }
                }
        }
    }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { launcher.launch(null) }) { Text("画像を分析") }
        labels.forEach { (label, confidence) ->
            ListItem(
                headlineContent = { Text(label) },
                trailingContent = { Text("${(confidence * 100).toInt()}%") }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `TextRecognition` | OCRテキスト認識 |
| `FaceDetection` | 顔検出+表情分析 |
| `BarcodeScanning` | バーコード/QR読取 |
| `ImageLabeling` | 画像分類 |

- ML Kitはオンデバイスで動作（ネット不要）
- `InputImage.fromBitmap()`で画像を入力
- 日本語OCRは`text-recognition-japanese`を使用
- 顔検出は表情・ランドマーク検出に対応

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-compose-camerax-2026)
- [Compose Permissions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permissions-2026)
- [Compose ImageLoading](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
