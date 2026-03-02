---
title: "Compose HapticFeedback完全ガイド — 触覚フィードバック/バイブレーション/振動パターン"
emoji: "📳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "haptic"]
published: true
---

## この記事で学べること

**Compose HapticFeedback**（触覚フィードバック、バイブレーション、振動パターン、Compose統合）を解説します。

---

## 基本HapticFeedback

```kotlin
@Composable
fun HapticDemo() {
    val haptic = LocalHapticFeedback.current

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Button(onClick = {
            haptic.performHapticFeedback(HapticFeedbackType.LongPress)
        }) { Text("長押しフィードバック") }

        Button(onClick = {
            haptic.performHapticFeedback(HapticFeedbackType.TextHandleMove)
        }) { Text("テキスト選択フィードバック") }
    }
}
```

---

## カスタムバイブレーション

```kotlin
@Composable
fun CustomVibration() {
    val context = LocalContext.current

    fun vibrate(durationMs: Long) {
        val vibrator = context.getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            vibrator.vibrate(VibrationEffect.createOneShot(durationMs, VibrationEffect.DEFAULT_AMPLITUDE))
        }
    }

    fun vibratePattern() {
        val vibrator = context.getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            // 待機50ms→振動100ms→待機50ms→振動200ms
            val pattern = longArrayOf(0, 100, 50, 200)
            vibrator.vibrate(VibrationEffect.createWaveform(pattern, -1))
        }
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Button(onClick = { vibrate(50) }) { Text("短い振動") }
        Button(onClick = { vibrate(200) }) { Text("長い振動") }
        Button(onClick = { vibratePattern() }) { Text("パターン振動") }
    }
}
```

---

## インタラクション連動

```kotlin
@Composable
fun HapticSwitch() {
    val haptic = LocalHapticFeedback.current
    var checked by remember { mutableStateOf(false) }

    Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
        Text("通知")
        Spacer(Modifier.weight(1f))
        Switch(
            checked = checked,
            onCheckedChange = {
                checked = it
                haptic.performHapticFeedback(HapticFeedbackType.LongPress)
            }
        )
    }
}

@Composable
fun HapticButton(onClick: () -> Unit, content: @Composable RowScope.() -> Unit) {
    val haptic = LocalHapticFeedback.current

    Button(onClick = {
        haptic.performHapticFeedback(HapticFeedbackType.LongPress)
        onClick()
    }, content = content)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LocalHapticFeedback` | Compose触覚 |
| `HapticFeedbackType` | フィードバック種別 |
| `VibrationEffect` | カスタム振動 |
| `createWaveform` | パターン振動 |

- `LocalHapticFeedback`でCompose内の触覚フィードバック
- ボタンタップ/スイッチ切替時に使用でUX向上
- `VibrationEffect`でカスタムパターン振動
- `VIBRATE`パーミッション不要（HapticFeedback経由）

---

8種類のAndroidアプリテンプレート（UX最適化対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose AccessibilityActions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-accessibility-actions-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
- [Compose Gesture](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-2026)
