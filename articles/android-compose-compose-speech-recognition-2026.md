---
title: "Compose SpeechRecognition完全ガイド — 音声認識/SpeechRecognizer/リアルタイム変換"
emoji: "🎙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "speech"]
published: true
---

## この記事で学べること

**Compose SpeechRecognition**（SpeechRecognizer、音声→テキスト変換、リアルタイム認識、Compose統合）を解説します。

---

## 基本音声認識

```kotlin
@Composable
fun SpeechRecognitionDemo() {
    val context = LocalContext.current
    var recognizedText by remember { mutableStateOf("") }
    var isListening by remember { mutableStateOf(false) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            val matches = result.data?.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS)
            recognizedText = matches?.firstOrNull() ?: ""
        }
        isListening = false
    }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        Text(recognizedText.ifEmpty { "マイクボタンを押して話してください" },
            style = MaterialTheme.typography.bodyLarge, modifier = Modifier.padding(16.dp))

        FloatingActionButton(onClick = {
            isListening = true
            val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
                putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
                putExtra(RecognizerIntent.EXTRA_LANGUAGE, "ja-JP")
                putExtra(RecognizerIntent.EXTRA_PROMPT, "話してください")
            }
            launcher.launch(intent)
        }) {
            Icon(if (isListening) Icons.Default.Stop else Icons.Default.Mic,
                if (isListening) "停止" else "録音")
        }
    }
}
```

---

## リアルタイム認識

```kotlin
@Composable
fun RealtimeSpeech() {
    val context = LocalContext.current
    var partialText by remember { mutableStateOf("") }
    var finalText by remember { mutableStateOf("") }
    var isListening by remember { mutableStateOf(false) }

    val recognizer = remember { SpeechRecognizer.createSpeechRecognizer(context) }

    DisposableEffect(Unit) {
        recognizer.setRecognitionListener(object : RecognitionListener {
            override fun onResults(results: Bundle) {
                val matches = results.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                finalText = matches?.firstOrNull() ?: ""
                partialText = ""
                isListening = false
            }
            override fun onPartialResults(partial: Bundle) {
                val matches = partial.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                partialText = matches?.firstOrNull() ?: ""
            }
            override fun onReadyForSpeech(params: Bundle) {}
            override fun onBeginningOfSpeech() {}
            override fun onRmsChanged(rmsdB: Float) {}
            override fun onBufferReceived(buffer: ByteArray) {}
            override fun onEndOfSpeech() {}
            override fun onError(error: Int) { isListening = false }
            override fun onEvent(eventType: Int, params: Bundle) {}
        })

        onDispose { recognizer.destroy() }
    }

    Column(Modifier.padding(16.dp)) {
        Text(finalText.ifEmpty { partialText.ifEmpty { "..." } },
            style = MaterialTheme.typography.bodyLarge)
        if (partialText.isNotEmpty()) Text("(認識中: $partialText)", color = Color.Gray)

        Button(onClick = {
            if (isListening) { recognizer.stopListening(); isListening = false }
            else {
                val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
                    putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
                    putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
                    putExtra(RecognizerIntent.EXTRA_LANGUAGE, "ja-JP")
                }
                recognizer.startListening(intent)
                isListening = true
            }
        }, Modifier.fillMaxWidth()) {
            Text(if (isListening) "停止" else "音声認識開始")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `RecognizerIntent` | 音声認識Intent |
| `SpeechRecognizer` | リアルタイム認識 |
| `EXTRA_PARTIAL_RESULTS` | 途中結果取得 |
| `onResults` | 最終結果 |

- `ACTION_RECOGNIZE_SPEECH`でシステムUI付き音声認識
- `SpeechRecognizer`でカスタムUIのリアルタイム認識
- `EXTRA_PARTIAL_RESULTS`で途中経過を表示
- `RECORD_AUDIO`パーミッションが必要

---

8種類のAndroidアプリテンプレート（音声機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TextToSpeech](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-to-speech-2026)
- [Compose PermissionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [Compose Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-2026)
