---
title: "Slider完全ガイド — Slider/RangeSlider/ステップ/カスタムThumb"
emoji: "🎚️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Slider**（Slider、RangeSlider、ステップ指定、カスタムThumb）を解説します。

---

## 基本Slider

```kotlin
@Composable
fun SliderExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // 基本
        var value by remember { mutableFloatStateOf(50f) }
        Text("値: ${value.toInt()}")
        Slider(
            value = value,
            onValueChange = { value = it },
            valueRange = 0f..100f
        )

        // ステップ付き
        var stepped by remember { mutableFloatStateOf(0f) }
        Text("ステップ: ${stepped.toInt()}")
        Slider(
            value = stepped,
            onValueChange = { stepped = it },
            valueRange = 0f..100f,
            steps = 9 // 10段階（0, 10, 20, ...100）
        )

        // 無効化
        Slider(value = 30f, onValueChange = {}, enabled = false)
    }
}
```

---

## RangeSlider

```kotlin
@Composable
fun RangeSliderExample() {
    var range by remember { mutableStateOf(20f..80f) }

    Column(Modifier.padding(16.dp)) {
        Text("範囲: ${range.start.toInt()} ~ ${range.endInclusive.toInt()}")
        RangeSlider(
            value = range,
            onValueChange = { range = it },
            valueRange = 0f..100f,
            steps = 19
        )
    }
}

// 価格フィルター
@Composable
fun PriceFilter() {
    var priceRange by remember { mutableStateOf(1000f..50000f) }
    val fmt = NumberFormat.getCurrencyInstance(Locale.JAPAN)

    Column(Modifier.padding(16.dp)) {
        Text("価格帯", style = MaterialTheme.typography.titleMedium)
        Text("${fmt.format(priceRange.start.toInt())} ~ ${fmt.format(priceRange.endInclusive.toInt())}")
        RangeSlider(
            value = priceRange,
            onValueChange = { priceRange = it },
            valueRange = 0f..100000f
        )
    }
}
```

---

## カスタムカラーSlider

```kotlin
@Composable
fun CustomSlider() {
    var volume by remember { mutableFloatStateOf(0.7f) }

    Column(Modifier.padding(16.dp)) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(Icons.Default.VolumeDown, null)
            Slider(
                value = volume,
                onValueChange = { volume = it },
                modifier = Modifier.weight(1f),
                colors = SliderDefaults.colors(
                    thumbColor = MaterialTheme.colorScheme.primary,
                    activeTrackColor = MaterialTheme.colorScheme.primary,
                    inactiveTrackColor = MaterialTheme.colorScheme.surfaceVariant
                )
            )
            Icon(Icons.Default.VolumeUp, null)
        }
        Text("${(volume * 100).toInt()}%", textAlign = TextAlign.Center, modifier = Modifier.fillMaxWidth())
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Slider` | 単一値スライダー |
| `RangeSlider` | 範囲スライダー |
| `steps` | ステップ刻み |
| `SliderDefaults.colors` | カラーカスタム |

- `Slider`で数値入力の直感的UI
- `RangeSlider`で価格帯等の範囲フィルター
- `steps`で離散値（10刻み等）の選択
- `SliderDefaults.colors`でテーマに合わせたカラー

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カラーピッカー](https://zenn.dev/myougatheaxo/articles/android-compose-compose-color-picker-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [State Management](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
