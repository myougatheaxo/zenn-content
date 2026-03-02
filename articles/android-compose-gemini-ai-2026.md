---
title: "Gemini AI + Android完全ガイド — on-device AI/テキスト生成/画像認識"
emoji: "🤖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ai"]
published: true
---

## この記事で学べること

**Gemini AI + Android**（Firebase AI SDK、テキスト生成、画像認識、チャットUI、ストリーミング）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.google.firebase:firebase-ai:16.0.0")
    implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
}
```

---

## テキスト生成

```kotlin
class GeminiRepository @Inject constructor() {
    private val model = Firebase.ai(backend = GenerativeBackend.googleAI())
        .generativeModel("gemini-2.0-flash")

    suspend fun generateText(prompt: String): String {
        val response = model.generateContent(prompt)
        return response.text ?: ""
    }

    fun generateTextStream(prompt: String): Flow<String> = flow {
        model.generateContentStream(prompt).collect { chunk ->
            chunk.text?.let { emit(it) }
        }
    }
}
```

---

## チャットUI

```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel = hiltViewModel()) {
    val messages by viewModel.messages.collectAsStateWithLifecycle()
    var input by remember { mutableStateOf("") }
    val isGenerating by viewModel.isGenerating.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize()) {
        LazyColumn(Modifier.weight(1f).padding(horizontal = 16.dp), reverseLayout = true) {
            items(messages.reversed()) { message ->
                ChatBubble(message)
            }
        }

        Row(Modifier.fillMaxWidth().padding(8.dp), verticalAlignment = Alignment.CenterVertically) {
            OutlinedTextField(
                value = input,
                onValueChange = { input = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("メッセージを入力") },
                enabled = !isGenerating
            )
            IconButton(
                onClick = {
                    viewModel.sendMessage(input)
                    input = ""
                },
                enabled = input.isNotBlank() && !isGenerating
            ) {
                if (isGenerating) CircularProgressIndicator(Modifier.size(24.dp))
                else Icon(Icons.AutoMirrored.Filled.Send, "送信")
            }
        }
    }
}

@Composable
fun ChatBubble(message: ChatMessage) {
    val isUser = message.role == "user"
    Row(
        Modifier.fillMaxWidth().padding(vertical = 4.dp),
        horizontalArrangement = if (isUser) Arrangement.End else Arrangement.Start
    ) {
        Card(
            colors = CardDefaults.cardColors(
                containerColor = if (isUser) MaterialTheme.colorScheme.primaryContainer
                    else MaterialTheme.colorScheme.surfaceVariant
            )
        ) {
            Text(message.text, Modifier.padding(12.dp), style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

---

## 画像認識

```kotlin
suspend fun analyzeImage(bitmap: Bitmap, prompt: String): String {
    val content = content {
        image(bitmap)
        text(prompt)
    }
    val response = model.generateContent(content)
    return response.text ?: ""
}

// Compose UI
@Composable
fun ImageAnalyzer(viewModel: GeminiViewModel = hiltViewModel()) {
    var selectedImage by remember { mutableStateOf<Bitmap?>(null) }
    val result by viewModel.analysisResult.collectAsStateWithLifecycle()

    val launcher = rememberLauncherForActivityResult(ActivityResultContracts.GetContent()) { uri ->
        uri?.let { /* load bitmap */ }
    }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { launcher.launch("image/*") }) { Text("画像を選択") }
        selectedImage?.let { bitmap ->
            Image(bitmap.asImageBitmap(), null, Modifier.fillMaxWidth().height(200.dp))
            Button(onClick = { viewModel.analyzeImage(bitmap, "この画像を説明してください") }) {
                Text("AI分析")
            }
        }
        result?.let { Text(it, Modifier.padding(top = 8.dp)) }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| テキスト生成 | `generateContent()` |
| ストリーミング | `generateContentStream()` |
| 画像認識 | `content { image() }` |
| チャット | `startChat()` |

- Firebase AI SDKでGeminiモデルを簡単利用
- ストリーミングでリアルタイム応答表示
- マルチモーダル対応（テキスト+画像）
- on-device推論で低レイテンシ

---

8種類のAndroidアプリテンプレート（AI連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ML Kit OCR](https://zenn.dev/myougatheaxo/articles/android-compose-mlkit-text-recognition-2026)
- [カメラ](https://zenn.dev/myougatheaxo/articles/android-compose-camera-capture-2026)
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
