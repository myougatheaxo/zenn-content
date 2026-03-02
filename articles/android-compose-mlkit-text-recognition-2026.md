---
title: "ML Kit テキスト認識完全ガイド — OCR + Compose"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "mlkit"]
published: true
---

## この記事で学べること

**ML Kit Text Recognition**（OCR、カメラ連携、日本語テキスト認識、結果表示）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    // ラテン文字
    implementation("com.google.mlkit:text-recognition:16.0.1")
    // 日本語・韓国語・中国語
    implementation("com.google.mlkit:text-recognition-japanese:16.0.1")

    // CameraX
    implementation("androidx.camera:camera-camera2:1.4.1")
    implementation("androidx.camera:camera-lifecycle:1.4.1")
    implementation("androidx.camera:camera-view:1.4.1")
}
```

---

## テキスト認識（画像から）

```kotlin
class TextRecognitionRepository @Inject constructor() {

    private val recognizer = TextRecognition.getClient(
        JapaneseTextRecognizerOptions.Builder().build()
    )

    suspend fun recognizeFromBitmap(bitmap: Bitmap): RecognizedText {
        val inputImage = InputImage.fromBitmap(bitmap, 0)
        return suspendCancellableCoroutine { cont ->
            recognizer.process(inputImage)
                .addOnSuccessListener { text ->
                    cont.resume(RecognizedText(
                        fullText = text.text,
                        blocks = text.textBlocks.map { block ->
                            TextBlock(
                                text = block.text,
                                boundingBox = block.boundingBox,
                                lines = block.lines.map { it.text }
                            )
                        }
                    ))
                }
                .addOnFailureListener { e ->
                    cont.resumeWithException(e)
                }
        }
    }

    suspend fun recognizeFromUri(context: Context, uri: Uri): RecognizedText {
        val inputImage = InputImage.fromFilePath(context, uri)
        return suspendCancellableCoroutine { cont ->
            recognizer.process(inputImage)
                .addOnSuccessListener { text ->
                    cont.resume(RecognizedText(
                        fullText = text.text,
                        blocks = text.textBlocks.map { block ->
                            TextBlock(block.text, block.boundingBox, block.lines.map { it.text })
                        }
                    ))
                }
                .addOnFailureListener { cont.resumeWithException(it) }
        }
    }
}

data class RecognizedText(val fullText: String, val blocks: List<TextBlock>)
data class TextBlock(val text: String, val boundingBox: Rect?, val lines: List<String>)
```

---

## CameraXリアルタイム認識

```kotlin
class TextAnalyzer(
    private val onTextDetected: (String) -> Unit
) : ImageAnalysis.Analyzer {

    private val recognizer = TextRecognition.getClient(
        JapaneseTextRecognizerOptions.Builder().build()
    )

    @OptIn(ExperimentalGetImage::class)
    override fun analyze(imageProxy: ImageProxy) {
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()
            return
        }

        val inputImage = InputImage.fromMediaImage(
            mediaImage, imageProxy.imageInfo.rotationDegrees
        )

        recognizer.process(inputImage)
            .addOnSuccessListener { text ->
                if (text.text.isNotEmpty()) {
                    onTextDetected(text.text)
                }
            }
            .addOnCompleteListener {
                imageProxy.close()
            }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun TextRecognitionScreen(viewModel: OcrViewModel = hiltViewModel()) {
    val recognizedText by viewModel.recognizedText.collectAsStateWithLifecycle()
    val isProcessing by viewModel.isProcessing.collectAsStateWithLifecycle()

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri ->
        uri?.let { viewModel.recognizeFromUri(it) }
    }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("テキスト認識（OCR）", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { launcher.launch("image/*") }) {
                Text("画像を選択")
            }
        }

        if (isProcessing) {
            Spacer(Modifier.height(16.dp))
            CircularProgressIndicator()
        }

        recognizedText?.let { result ->
            Spacer(Modifier.height(16.dp))
            Card(Modifier.fillMaxWidth()) {
                Column(Modifier.padding(16.dp)) {
                    Text("認識結果", style = MaterialTheme.typography.titleMedium)
                    Spacer(Modifier.height(8.dp))
                    SelectionContainer {
                        Text(result.fullText)
                    }
                }
            }

            Spacer(Modifier.height(8.dp))

            // コピーボタン
            val clipboardManager = LocalClipboardManager.current
            OutlinedButton(onClick = {
                clipboardManager.setText(AnnotatedString(result.fullText))
            }) {
                Text("コピー")
            }
        }
    }
}
```

---

## まとめ

| 機能 | クラス |
|------|--------|
| テキスト認識 | `TextRecognizer` |
| 日本語対応 | `JapaneseTextRecognizerOptions` |
| 画像入力 | `InputImage.fromBitmap/fromFilePath` |
| カメラ連携 | `ImageAnalysis.Analyzer` |

- ML Kitはオンデバイスで高速OCR
- 日本語認識は専用モジュール追加
- CameraX連携でリアルタイム認識
- `SelectionContainer`で認識結果をコピー可能に

---

8種類のAndroidアプリテンプレート（ML Kit対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [バーコードスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-barcode-scanner-2026)
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camera-capture-2026)
- [クリップボード](https://zenn.dev/myougatheaxo/articles/android-compose-clipboard-copy-2026)
