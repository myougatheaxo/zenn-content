---
title: "音声入力完全ガイド — SpeechRecognizer/リアルタイム文字起こし/多言語対応"
emoji: "🎙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "speech"]
published: true
---

## この記事で学べること

**音声入力**（SpeechRecognizer、リアルタイム文字起こし、多言語対応、Compose UI連携）を解説します。

---

## SpeechRecognizer

```kotlin
class SpeechRecognizerHelper(private val context: Context) {
    private var recognizer: SpeechRecognizer? = null
    private val _result = MutableStateFlow("")
    val result: StateFlow<String> = _result.asStateFlow()
    private val _isListening = MutableStateFlow(false)
    val isListening: StateFlow<Boolean> = _isListening.asStateFlow()

    fun startListening(language: String = "ja-JP") {
        recognizer = SpeechRecognizer.createSpeechRecognizer(context)
        recognizer?.setRecognitionListener(object : RecognitionListener {
            override fun onResults(results: Bundle?) {
                val matches = results?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                _result.value = matches?.firstOrNull() ?: ""
                _isListening.value = false
            }

            override fun onPartialResults(partialResults: Bundle?) {
                val partial = partialResults?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                _result.value = partial?.firstOrNull() ?: ""
            }

            override fun onError(error: Int) {
                _isListening.value = false
            }

            override fun onReadyForSpeech(params: Bundle?) { _isListening.value = true }
            override fun onBeginningOfSpeech() {}
            override fun onRmsChanged(rmsdB: Float) {}
            override fun onBufferReceived(buffer: ByteArray?) {}
            override fun onEndOfSpeech() {}
            override fun onEvent(eventType: Int, params: Bundle?) {}
        })

        val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
            putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
            putExtra(RecognizerIntent.EXTRA_LANGUAGE, language)
            putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
        }
        recognizer?.startListening(intent)
    }

    fun stopListening() {
        recognizer?.stopListening()
        _isListening.value = false
    }

    fun destroy() {
        recognizer?.destroy()
        recognizer = null
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun SpeechInputScreen() {
    val context = LocalContext.current
    val helper = remember { SpeechRecognizerHelper(context) }
    val result by helper.result.collectAsStateWithLifecycle("")
    val isListening by helper.isListening.collectAsStateWithLifecycle(false)

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) helper.startListening()
    }

    DisposableEffect(Unit) {
        onDispose { helper.destroy() }
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // 認識結果表示
        Card(Modifier.fillMaxWidth().weight(1f)) {
            Text(
                text = result.ifEmpty { "音声入力の結果がここに表示されます" },
                modifier = Modifier.padding(16.dp),
                color = if (result.isEmpty()) Color.Gray else LocalContentColor.current
            )
        }

        Spacer(Modifier.height(24.dp))

        // マイクボタン
        FloatingActionButton(
            onClick = {
                if (isListening) helper.stopListening()
                else permissionLauncher.launch(Manifest.permission.RECORD_AUDIO)
            },
            containerColor = if (isListening) MaterialTheme.colorScheme.error
                else MaterialTheme.colorScheme.primary
        ) {
            Icon(if (isListening) Icons.Default.Stop else Icons.Default.Mic, "音声入力")
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 音声認識 | `SpeechRecognizer` |
| 部分結果 | `EXTRA_PARTIAL_RESULTS` |
| 言語指定 | `EXTRA_LANGUAGE` |
| パーミッション | `RECORD_AUDIO` |

- `SpeechRecognizer`でオフライン/オンライン音声認識
- `EXTRA_PARTIAL_RESULTS`でリアルタイム文字起こし
- `RECORD_AUDIO`パーミッション必須
- 多言語対応で日本語/英語切替

---

8種類のAndroidアプリテンプレート（音声機能対応可）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Text-to-Speech](https://zenn.dev/myougatheaxo/articles/android-compose-text-to-speech-2026)
- [音声録音](https://zenn.dev/myougatheaxo/articles/android-compose-audio-recorder-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-2026)
