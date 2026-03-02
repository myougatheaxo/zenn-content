---
title: "Slider/RangeSlider完全ガイド — カスタムデザイン/ステップ/ラベル"
emoji: "🎚️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Slider/RangeSlider**（基本スライダー、範囲選択、カスタムデザイン、ステップ、ラベル表示）を解説します。

---

## 基本Slider

```kotlin
@Composable
fun BasicSlider() {
    var value by remember { mutableFloatStateOf(0.5f) }

    Column(Modifier.padding(16.dp)) {
        Text("音量: ${(value * 100).toInt()}%")
        Slider(
            value = value,
            onValueChange = { value = it },
            valueRange = 0f..1f,
            colors = SliderDefaults.colors(
                thumbColor = MaterialTheme.colorScheme.primary,
                activeTrackColor = MaterialTheme.colorScheme.primary,
                inactiveTrackColor = MaterialTheme.colorScheme.surfaceVariant
            )
        )
    }
}

// ステップ付きSlider
@Composable
fun SteppedSlider() {
    var value by remember { mutableFloatStateOf(3f) }

    Column(Modifier.padding(16.dp)) {
        Text("評価: ${value.toInt()}/5")
        Slider(
            value = value,
            onValueChange = { value = it },
            valueRange = 1f..5f,
            steps = 3  // 1,2,3,4,5 の5段階（中間ステップ数）
        )
    }
}
```

---

## RangeSlider

```kotlin
@Composable
fun PriceRangeSlider() {
    var range by remember { mutableStateOf(1000f..5000f) }

    Column(Modifier.padding(16.dp)) {
        Text("価格帯: ¥${range.start.toInt()} - ¥${range.endInclusive.toInt()}")
        Spacer(Modifier.height(8.dp))

        RangeSlider(
            value = range,
            onValueChange = { range = it },
            valueRange = 0f..10000f,
            steps = 9,  // 1000円刻み
            colors = SliderDefaults.colors(
                thumbColor = MaterialTheme.colorScheme.primary,
                activeTrackColor = MaterialTheme.colorScheme.primary
            )
        )

        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
            Text("¥0", style = MaterialTheme.typography.labelSmall)
            Text("¥10,000", style = MaterialTheme.typography.labelSmall)
        }
    }
}
```

---

## カスタムデザインSlider

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CustomSlider() {
    var value by remember { mutableFloatStateOf(50f) }

    Column(Modifier.padding(16.dp)) {
        Slider(
            value = value,
            onValueChange = { value = it },
            valueRange = 0f..100f,
            thumb = {
                Box(
                    Modifier
                        .size(28.dp)
                        .shadow(4.dp, CircleShape)
                        .background(MaterialTheme.colorScheme.primary, CircleShape),
                    contentAlignment = Alignment.Center
                ) {
                    Text(
                        "${value.toInt()}",
                        color = MaterialTheme.colorScheme.onPrimary,
                        fontSize = 10.sp,
                        fontWeight = FontWeight.Bold
                    )
                }
            },
            track = { sliderState ->
                SliderDefaults.Track(
                    sliderState = sliderState,
                    modifier = Modifier.height(8.dp)
                )
            }
        )
    }
}
```

---

## ラベル付きSlider

```kotlin
@Composable
fun LabeledSlider(
    label: String,
    value: Float,
    onValueChange: (Float) -> Unit,
    valueRange: ClosedFloatingPointRange<Float>,
    format: (Float) -> String = { it.toInt().toString() }
) {
    Column(Modifier.padding(vertical = 8.dp)) {
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Text(label, style = MaterialTheme.typography.labelLarge)
            Text(
                format(value),
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.primary,
                fontWeight = FontWeight.Bold
            )
        }
        Slider(value = value, onValueChange = onValueChange, valueRange = valueRange)
    }
}

// 使用
@Composable
fun SettingsSliders() {
    var brightness by remember { mutableFloatStateOf(80f) }
    var fontSize by remember { mutableFloatStateOf(16f) }

    Column(Modifier.padding(16.dp)) {
        LabeledSlider("明るさ", brightness, { brightness = it }, 0f..100f) { "${it.toInt()}%" }
        LabeledSlider("文字サイズ", fontSize, { fontSize = it }, 12f..24f) { "${it.toInt()}sp" }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|---------------|------|
| `Slider` | 単一値選択 |
| `RangeSlider` | 範囲選択 |
| `steps` | 離散値 |
| カスタムthumb | デザイン自由 |

- `Slider`で連続値/ステップ値の入力
- `RangeSlider`で価格帯などの範囲指定
- カスタムthumbで値をサム上に表示
- ラベル付きコンポーネントで再利用性UP

---

8種類のAndroidアプリテンプレート（カスタムUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [フォーム入力](https://zenn.dev/myougatheaxo/articles/android-compose-form-validation-2026)
- [状態管理](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
