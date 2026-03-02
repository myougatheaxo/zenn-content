---
title: "Slider・Stepper実装ガイド — Composeで数値入力UIを作る"
emoji: "🎚️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**Slider**と**カスタムStepper**で数値入力UIを実装する方法を解説します。

---

## 基本のSlider

```kotlin
@Composable
fun BasicSlider() {
    var value by remember { mutableFloatStateOf(0.5f) }

    Column(Modifier.padding(16.dp)) {
        Text("音量: ${(value * 100).toInt()}%")
        Slider(
            value = value,
            onValueChange = { value = it }
        )
    }
}
```

---

## ステップ付きSlider

```kotlin
@Composable
fun SteppedSlider() {
    var rating by remember { mutableFloatStateOf(3f) }

    Column(Modifier.padding(16.dp)) {
        Text("評価: ${rating.toInt()} / 5")
        Slider(
            value = rating,
            onValueChange = { rating = it },
            valueRange = 1f..5f,
            steps = 3  // 1, 2, 3, 4, 5 の5段階
        )
    }
}
```

---

## RangeSlider（範囲選択）

```kotlin
@Composable
fun PriceRangeSlider() {
    var range by remember { mutableStateOf(1000f..5000f) }

    Column(Modifier.padding(16.dp)) {
        Text("価格帯: ¥${range.start.toInt()} 〜 ¥${range.endInclusive.toInt()}")
        RangeSlider(
            value = range,
            onValueChange = { range = it },
            valueRange = 0f..10000f,
            steps = 9
        )
    }
}
```

---

## カスタムStepper（+/-ボタン）

```kotlin
@Composable
fun Stepper(
    value: Int,
    onValueChange: (Int) -> Unit,
    range: IntRange = 0..99
) {
    Row(
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        FilledIconButton(
            onClick = { if (value > range.first) onValueChange(value - 1) },
            enabled = value > range.first
        ) {
            Icon(Icons.Default.Remove, "減らす")
        }

        Text(
            "$value",
            style = MaterialTheme.typography.headlineSmall,
            modifier = Modifier.widthIn(min = 48.dp),
            textAlign = TextAlign.Center
        )

        FilledIconButton(
            onClick = { if (value < range.last) onValueChange(value + 1) },
            enabled = value < range.last
        ) {
            Icon(Icons.Default.Add, "増やす")
        }
    }
}

// 使用例
@Composable
fun QuantitySelector() {
    var quantity by remember { mutableIntStateOf(1) }

    Column(Modifier.padding(16.dp)) {
        Text("数量")
        Stepper(
            value = quantity,
            onValueChange = { quantity = it },
            range = 1..10
        )
    }
}
```

---

## カスタムカラーSlider

```kotlin
Slider(
    value = value,
    onValueChange = { value = it },
    colors = SliderDefaults.colors(
        thumbColor = MaterialTheme.colorScheme.primary,
        activeTrackColor = MaterialTheme.colorScheme.primary,
        inactiveTrackColor = MaterialTheme.colorScheme.surfaceVariant
    )
)
```

---

## まとめ

- `Slider`で連続値入力
- `steps`パラメータで段階的選択
- `RangeSlider`で範囲選択（価格帯など）
- カスタムStepperは`Row` + `IconButton`で構築
- `SliderDefaults.colors`でカスタマイズ

---

8種類のAndroidアプリテンプレート（数値入力UI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [状態管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
