---
title: "Compose TensorFlow Lite完全ガイド — オンデバイスML/画像分類/物体検出"
emoji: "🧠"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "tensorflow"]
published: true
---

## この記事で学べること

**Compose TensorFlow Lite**（オンデバイスML、画像分類、物体検出、モデル読み込み）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("org.tensorflow:tensorflow-lite:2.14.0")
    implementation("org.tensorflow:tensorflow-lite-support:0.4.4")
    implementation("org.tensorflow:tensorflow-lite-task-vision:0.4.4")
}

android {
    aaptOptions {
        noCompress("tflite")
    }
}
```

---

## 画像分類

```kotlin
class ImageClassifier(private val context: Context) {
    private val classifier: ImageClassifier by lazy {
        val options = ImageClassifier.ImageClassifierOptions.builder()
            .setMaxResults(5)
            .setScoreThreshold(0.3f)
            .build()
        ImageClassifier.createFromFileAndOptions(context, "model.tflite", options)
    }

    fun classify(bitmap: Bitmap): List<Pair<String, Float>> {
        val image = TensorImage.fromBitmap(bitmap)
        val results = classifier.classify(image)
        return results.flatMap { it.categories }
            .map { it.label to it.score }
            .sortedByDescending { it.second }
    }
}

@Composable
fun ClassifierScreen() {
    val context = LocalContext.current
    val classifier = remember { ImageClassifier(context) }
    var results by remember { mutableStateOf<List<Pair<String, Float>>>(emptyList()) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicturePreview()
    ) { bitmap ->
        bitmap?.let { results = classifier.classify(it) }
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Button(onClick = { launcher.launch(null) }) { Text("写真を撮影して分類") }

        results.forEach { (label, score) ->
            Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                Text(label)
                Text("${(score * 100).toInt()}%", fontWeight = FontWeight.Bold)
            }
            LinearProgressIndicator(progress = { score }, Modifier.fillMaxWidth())
        }
    }
}
```

---

## 物体検出

```kotlin
class ObjectDetectorHelper(private val context: Context) {
    private val detector: ObjectDetector by lazy {
        val options = ObjectDetector.ObjectDetectorOptions.builder()
            .setMaxResults(10)
            .setScoreThreshold(0.5f)
            .build()
        ObjectDetector.createFromFileAndOptions(context, "detect.tflite", options)
    }

    fun detect(bitmap: Bitmap): List<Detection> {
        val image = TensorImage.fromBitmap(bitmap)
        val results = detector.detect(image)
        return results
    }
}

@Composable
fun DetectionResultScreen(detections: List<Detection>) {
    LazyColumn(Modifier.padding(16.dp)) {
        items(detections) { detection ->
            val category = detection.categories.firstOrNull()
            ListItem(
                headlineContent = { Text(category?.label ?: "不明") },
                supportingContent = {
                    Text("信頼度: ${((category?.score ?: 0f) * 100).toInt()}%")
                },
                leadingContent = { Icon(Icons.Default.CropFree, null) }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ImageClassifier` | 画像分類 |
| `ObjectDetector` | 物体検出 |
| `TensorImage` | 画像変換 |
| `noCompress("tflite")` | モデルファイル非圧縮 |

- TensorFlow Liteでオンデバイスに推論（オフライン対応）
- `assets/`に`.tflite`モデルファイルを配置
- `noCompress`でモデルファイルの圧縮を回避
- Task APIで簡単に画像分類・物体検出を実装

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ML Kit](https://zenn.dev/myougatheaxo/articles/android-compose-compose-mlkit-2026)
- [Compose CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-compose-camerax-2026)
- [Compose Coil](https://zenn.dev/myougatheaxo/articles/android-compose-compose-coil-2026)
