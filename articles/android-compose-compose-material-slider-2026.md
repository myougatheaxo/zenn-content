---
title: "Compose Slider完全ガイド — M3 Slider/RangeSlider/ステップ/カスタムUI"
emoji: "🎚️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Slider**（M3 Slider、RangeSlider、ステップ設定、カスタムトラック）を解説します。

---

## 基本Slider

```kotlin
@Composable
fun SliderDemo() {
    var volume by remember { mutableFloatStateOf(0.5f) }

    Column(Modifier.padding(16.dp)) {
        Text("音量: ${(volume * 100).toInt()}%")
        Slider(value = volume, onValueChange = { volume = it })
    }
}

// ステップ付き
@Composable
fun StepSlider() {
    var rating by remember { mutableFloatStateOf(3f) }

    Column(Modifier.padding(16.dp)) {
        Text("評価: ${rating.toInt()}/5")
        Slider(value = rating, onValueChange = { rating = it },
            valueRange = 1f..5f, steps = 3) // 1, 2, 3, 4, 5
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

## ラベル付き

```kotlin
@Composable
fun LabeledSlider() {
    var value by remember { mutableFloatStateOf(50f) }

    Column(Modifier.padding(16.dp)) {
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
            Text("0%"); Text("100%")
        }
        Slider(
            value = value,
            onValueChange = { value = it },
            valueRange = 0f..100f,
            colors = SliderDefaults.colors(
                thumbColor = MaterialTheme.colorScheme.primary,
                activeTrackColor = MaterialTheme.colorScheme.primary,
                inactiveTrackColor = MaterialTheme.colorScheme.primaryContainer
            )
        )
        Text("${value.toInt()}%", Modifier.fillMaxWidth(), textAlign = TextAlign.Center,
            style = MaterialTheme.typography.titleMedium)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Slider` | 単一値スライダー |
| `RangeSlider` | 範囲スライダー |
| `steps` | ステップ数 |
| `SliderDefaults.colors` | カラー設定 |

- `Slider`で連続値またはステップ値の選択
- `RangeSlider`で価格帯等の範囲指定
- `steps`パラメータで離散的な値選択
- `SliderDefaults.colors`でブランドカラーに統一

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Switch](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-switch-2026)
- [Compose SettingsScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-settings-screen-2026)
- [Compose MaterialProgress](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-progress-2026)
