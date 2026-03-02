---
title: "Compose TextToSpeech完全ガイド — TTS/音声読み上げ/速度制御/言語切替"
emoji: "🗣️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "tts"]
published: true
---

## この記事で学べること

**Compose TextToSpeech**（TTS音声読み上げ、速度/ピッチ制御、言語切替、Compose統合）を解説します。

---

## 基本TTS

```kotlin
@Composable
fun TextToSpeechDemo() {
    val context = LocalContext.current
    var ttsReady by remember { mutableStateOf(false) }
    val tts = remember {
        TextToSpeech(context) { status ->
            if (status == TextToSpeech.SUCCESS) {
                ttsReady = true
            }
        }.apply {
            language = Locale.JAPANESE
        }
    }

    DisposableEffect(Unit) {
        onDispose { tts.shutdown() }
    }

    var text by remember { mutableStateOf("こんにちは、みょうがです") }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(value = text, onValueChange = { text = it },
            label = { Text("読み上げテキスト") }, modifier = Modifier.fillMaxWidth())

        Button(
            onClick = { tts.speak(text, TextToSpeech.QUEUE_FLUSH, null, "utterance_id") },
            enabled = ttsReady, modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.VolumeUp, null); Spacer(Modifier.width(8.dp)); Text("読み上げ")
        }

        OutlinedButton(onClick = { tts.stop() }, Modifier.fillMaxWidth()) {
            Text("停止")
        }
    }
}
```

---

## 速度・ピッチ制御

```kotlin
@Composable
fun TTSWithControls() {
    val context = LocalContext.current
    val tts = remember { TextToSpeech(context) { }.apply { language = Locale.JAPANESE } }
    var speed by remember { mutableFloatStateOf(1.0f) }
    var pitch by remember { mutableFloatStateOf(1.0f) }
    var text by remember { mutableStateOf("テスト読み上げ") }

    DisposableEffect(Unit) { onDispose { tts.shutdown() } }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(value = text, onValueChange = { text = it }, Modifier.fillMaxWidth())

        Text("速度: ${"%.1f".format(speed)}x")
        Slider(value = speed, onValueChange = { speed = it; tts.setSpeechRate(it) },
            valueRange = 0.5f..2.0f)

        Text("ピッチ: ${"%.1f".format(pitch)}")
        Slider(value = pitch, onValueChange = { pitch = it; tts.setPitch(it) },
            valueRange = 0.5f..2.0f)

        Button(onClick = { tts.speak(text, TextToSpeech.QUEUE_FLUSH, null, null) },
            Modifier.fillMaxWidth()) { Text("読み上げ") }
    }
}
```

---

## 読み上げ進捗

```kotlin
@Composable
fun TTSWithProgress() {
    val context = LocalContext.current
    var isSpeaking by remember { mutableStateOf(false) }
    val tts = remember {
        TextToSpeech(context) { }.apply {
            language = Locale.JAPANESE
            setOnUtteranceProgressListener(object : UtteranceProgressListener() {
                override fun onStart(utteranceId: String?) { isSpeaking = true }
                override fun onDone(utteranceId: String?) { isSpeaking = false }
                override fun onError(utteranceId: String?) { isSpeaking = false }
            })
        }
    }

    DisposableEffect(Unit) { onDispose { tts.shutdown() } }

    Button(onClick = {
        if (isSpeaking) tts.stop()
        else tts.speak("読み上げテスト", TextToSpeech.QUEUE_FLUSH, null, "id")
    }) {
        Icon(if (isSpeaking) Icons.Default.Stop else Icons.Default.VolumeUp, null)
        Spacer(Modifier.width(8.dp))
        Text(if (isSpeaking) "停止" else "読み上げ")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `TextToSpeech` | TTS初期化 |
| `speak()` | 読み上げ実行 |
| `setSpeechRate` | 速度設定 |
| `setPitch` | ピッチ設定 |

- `TextToSpeech`初期化はコールバック完了後に使用
- `DisposableEffect`で`shutdown()`を確実に呼ぶ
- `QUEUE_FLUSH`で前の読み上げを中断、`QUEUE_ADD`で追加
- `UtteranceProgressListener`で読み上げ状態を監視

---

8種類のAndroidアプリテンプレート（音声機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose SpeechRecognition](https://zenn.dev/myougatheaxo/articles/android-compose-compose-speech-recognition-2026)
- [Compose HapticFeedback](https://zenn.dev/myougatheaxo/articles/android-compose-compose-haptic-feedback-2026)
- [Compose Accessibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-accessibility-actions-2026)
