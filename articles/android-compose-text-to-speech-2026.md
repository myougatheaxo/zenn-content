---
title: "Text-to-Speech完全ガイド — TextToSpeech/音声合成/速度制御/言語切替"
emoji: "🗣️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "tts"]
published: true
---

## この記事で学べること

**Text-to-Speech**（TextToSpeech初期化、音声合成、速度/ピッチ制御、日本語対応）を解説します。

---

## TTS初期化

```kotlin
class TtsHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private var tts: TextToSpeech? = null
    private val _isReady = MutableStateFlow(false)
    val isReady: StateFlow<Boolean> = _isReady.asStateFlow()

    fun initialize() {
        tts = TextToSpeech(context) { status ->
            if (status == TextToSpeech.SUCCESS) {
                tts?.language = Locale.JAPANESE
                _isReady.value = true
            }
        }
    }

    fun speak(text: String, speed: Float = 1.0f, pitch: Float = 1.0f) {
        tts?.apply {
            setSpeechRate(speed)
            setPitch(pitch)
            speak(text, TextToSpeech.QUEUE_FLUSH, null, "utterance_${System.currentTimeMillis()}")
        }
    }

    fun stop() { tts?.stop() }

    fun shutdown() {
        tts?.shutdown()
        tts = null
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun TtsScreen(viewModel: TtsViewModel = hiltViewModel()) {
    val isReady by viewModel.isReady.collectAsStateWithLifecycle(false)
    var text by remember { mutableStateOf("こんにちは、これはテスト音声です。") }
    var speed by remember { mutableFloatStateOf(1.0f) }
    var pitch by remember { mutableFloatStateOf(1.0f) }

    DisposableEffect(Unit) {
        viewModel.initialize()
        onDispose { viewModel.shutdown() }
    }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("読み上げテキスト") },
            modifier = Modifier.fillMaxWidth(),
            minLines = 3
        )

        Spacer(Modifier.height(16.dp))

        Text("速度: ${"%.1f".format(speed)}x")
        Slider(value = speed, onValueChange = { speed = it }, valueRange = 0.5f..2.0f)

        Text("ピッチ: ${"%.1f".format(pitch)}x")
        Slider(value = pitch, onValueChange = { pitch = it }, valueRange = 0.5f..2.0f)

        Spacer(Modifier.height(16.dp))

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(
                onClick = { viewModel.speak(text, speed, pitch) },
                enabled = isReady
            ) {
                Icon(Icons.Default.VolumeUp, null)
                Spacer(Modifier.width(8.dp))
                Text("読み上げ")
            }
            OutlinedButton(onClick = { viewModel.stop() }) { Text("停止") }
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 初期化 | `TextToSpeech(context)` |
| 読み上げ | `speak()` |
| 速度 | `setSpeechRate()` |
| ピッチ | `setPitch()` |

- `TextToSpeech`でオフライン音声合成
- `Locale.JAPANESE`で日本語読み上げ
- 速度/ピッチをスライダーで調整
- `shutdown()`でリソース解放必須

---

8種類のAndroidアプリテンプレート（音声機能対応可）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [音声認識](https://zenn.dev/myougatheaxo/articles/android-compose-speech-recognition-2026)
- [Media3](https://zenn.dev/myougatheaxo/articles/android-compose-media3-exoplayer-2026)
- [アクセシビリティ](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-2026)
