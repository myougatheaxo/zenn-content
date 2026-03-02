---
title: "音声認識完全ガイド — SpeechRecognizer/TTS/Compose連携"
emoji: "🎤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "speech"]
published: true
---

## この記事で学べること

**音声認識**（SpeechRecognizer、Text-to-Speech、音声入力UI、Compose連携）を解説します。

---

## 音声認識をFlowに変換

```kotlin
class SpeechRecognizerHelper(private val context: Context) {
    private val _results = MutableSharedFlow<SpeechResult>(replay = 1)
    val results: SharedFlow<SpeechResult> = _results

    private var recognizer: SpeechRecognizer? = null

    fun startListening(language: String = "ja-JP") {
        recognizer = SpeechRecognizer.createSpeechRecognizer(context).apply {
            setRecognitionListener(object : RecognitionListener {
                override fun onResults(results: Bundle?) {
                    val matches = results?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                    matches?.firstOrNull()?.let {
                        _results.tryEmit(SpeechResult.Success(it))
                    }
                }

                override fun onPartialResults(partialResults: Bundle?) {
                    val partial = partialResults?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                    partial?.firstOrNull()?.let {
                        _results.tryEmit(SpeechResult.Partial(it))
                    }
                }

                override fun onError(error: Int) {
                    _results.tryEmit(SpeechResult.Error(error))
                }

                override fun onReadyForSpeech(params: Bundle?) {}
                override fun onBeginningOfSpeech() {}
                override fun onRmsChanged(rmsdB: Float) {}
                override fun onBufferReceived(buffer: ByteArray?) {}
                override fun onEndOfSpeech() {}
                override fun onEvent(eventType: Int, params: Bundle?) {}
            })
        }

        val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
            putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
            putExtra(RecognizerIntent.EXTRA_LANGUAGE, language)
            putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
        }

        recognizer?.startListening(intent)
    }

    fun stopListening() {
        recognizer?.stopListening()
        recognizer?.destroy()
        recognizer = null
    }
}

sealed interface SpeechResult {
    data class Success(val text: String) : SpeechResult
    data class Partial(val text: String) : SpeechResult
    data class Error(val code: Int) : SpeechResult
}
```

---

## Text-to-Speech

```kotlin
class TtsHelper(context: Context) {
    private var tts: TextToSpeech? = null
    private var isReady = false

    init {
        tts = TextToSpeech(context) { status ->
            if (status == TextToSpeech.SUCCESS) {
                tts?.language = Locale.JAPAN
                isReady = true
            }
        }
    }

    fun speak(text: String) {
        if (isReady) {
            tts?.speak(text, TextToSpeech.QUEUE_FLUSH, null, "utterance_id")
        }
    }

    fun stop() { tts?.stop() }
    fun shutdown() { tts?.shutdown() }
}
```

---

## Compose画面

```kotlin
@Composable
fun VoiceInputScreen() {
    val context = LocalContext.current
    val speechHelper = remember { SpeechRecognizerHelper(context) }
    val ttsHelper = remember { TtsHelper(context) }
    var recognizedText by remember { mutableStateOf("") }
    var isListening by remember { mutableStateOf(false) }

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) { speechHelper.startListening(); isListening = true }
    }

    LaunchedEffect(Unit) {
        speechHelper.results.collect { result ->
            when (result) {
                is SpeechResult.Success -> { recognizedText = result.text; isListening = false }
                is SpeechResult.Partial -> recognizedText = result.text
                is SpeechResult.Error -> isListening = false
            }
        }
    }

    DisposableEffect(Unit) {
        onDispose { speechHelper.stopListening(); ttsHelper.shutdown() }
    }

    Column(Modifier.fillMaxSize().padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        Text("音声入力", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(24.dp))

        IconButton(
            onClick = {
                if (isListening) { speechHelper.stopListening(); isListening = false }
                else permissionLauncher.launch(Manifest.permission.RECORD_AUDIO)
            },
            modifier = Modifier.size(80.dp)
        ) {
            Icon(
                if (isListening) Icons.Default.Stop else Icons.Default.Mic,
                "音声入力",
                modifier = Modifier.size(48.dp),
                tint = if (isListening) MaterialTheme.colorScheme.error else MaterialTheme.colorScheme.primary
            )
        }

        if (recognizedText.isNotEmpty()) {
            Card(Modifier.fillMaxWidth().padding(top = 16.dp)) {
                Text(recognizedText, Modifier.padding(16.dp))
            }
            Button(onClick = { ttsHelper.speak(recognizedText) }) { Text("読み上げ") }
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 音声認識 | `SpeechRecognizer` |
| テキスト読み上げ | `TextToSpeech` |
| 部分結果 | `EXTRA_PARTIAL_RESULTS` |
| 言語設定 | `EXTRA_LANGUAGE` |

- `SpeechRecognizer`でオンデバイス音声認識
- `TextToSpeech`でテキスト読み上げ
- `RECORD_AUDIO`パーミッション必須
- 部分結果でリアルタイム表示

---

8種類のAndroidアプリテンプレート（音声機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ML Kit OCR](https://zenn.dev/myougatheaxo/articles/android-compose-mlkit-text-recognition-2026)
- [Gemini AI](https://zenn.dev/myougatheaxo/articles/android-compose-gemini-ai-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
