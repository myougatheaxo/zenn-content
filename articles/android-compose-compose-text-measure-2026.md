---
title: "TextMeasurer完全ガイド — テキスト計測/drawText/動的レイアウト"
emoji: "📏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

**TextMeasurer**（テキスト計測、Canvas上のdrawText、動的レイアウト計算）を解説します。

---

## rememberTextMeasurer

```kotlin
@Composable
fun TextMeasurerBasic() {
    val textMeasurer = rememberTextMeasurer()
    val text = "Hello, Compose!"

    Canvas(Modifier.fillMaxWidth().height(100.dp)) {
        val textLayoutResult = textMeasurer.measure(
            text = AnnotatedString(text),
            style = TextStyle(fontSize = 24.sp, color = Color.Black)
        )
        drawText(textLayoutResult)
    }
}
```

---

## テキストサイズ計測

```kotlin
@Composable
fun MeasuredTextBox() {
    val textMeasurer = rememberTextMeasurer()
    val style = MaterialTheme.typography.bodyLarge

    Canvas(Modifier.fillMaxWidth().height(120.dp)) {
        val result = textMeasurer.measure(
            text = AnnotatedString("計測されたテキスト"),
            style = style.copy(color = Color.Black)
        )
        val textSize = result.size

        // テキストに合わせた背景
        drawRoundRect(
            color = Color(0xFFE8F5E9),
            size = Size(
                textSize.width + 32.dp.toPx(),
                textSize.height + 16.dp.toPx()
            ),
            cornerRadius = CornerRadius(8.dp.toPx())
        )
        drawText(
            result,
            topLeft = Offset(16.dp.toPx(), 8.dp.toPx())
        )
    }
}

// 動的幅調整
@Composable
fun DynamicWidthChip(label: String) {
    val textMeasurer = rememberTextMeasurer()
    val style = TextStyle(fontSize = 14.sp, color = Color.White)

    Canvas(
        Modifier.height(32.dp).width(
            with(LocalDensity.current) {
                val w = textMeasurer.measure(AnnotatedString(label), style).size.width
                (w + 24.dp.roundToPx()).toDp()
            }
        )
    ) {
        drawRoundRect(Color(0xFF1976D2), cornerRadius = CornerRadius(16.dp.toPx()))
        val result = textMeasurer.measure(AnnotatedString(label), style)
        drawText(result, topLeft = Offset(12.dp.toPx(), (size.height - result.size.height) / 2))
    }
}
```

---

## チャートラベル

```kotlin
@Composable
fun BarChartWithLabels(data: List<Pair<String, Float>>) {
    val textMeasurer = rememberTextMeasurer()
    val maxValue = data.maxOf { it.second }

    Canvas(Modifier.fillMaxWidth().height(200.dp)) {
        val barWidth = size.width / (data.size * 2)

        data.forEachIndexed { index, (label, value) ->
            val barHeight = (value / maxValue) * (size.height - 40.dp.toPx())
            val x = barWidth * (index * 2 + 0.5f)

            drawRect(
                color = Color(0xFF42A5F5),
                topLeft = Offset(x, size.height - 40.dp.toPx() - barHeight),
                size = Size(barWidth, barHeight)
            )

            val textResult = textMeasurer.measure(
                AnnotatedString(label),
                style = TextStyle(fontSize = 10.sp, color = Color.Gray)
            )
            drawText(
                textResult,
                topLeft = Offset(
                    x + (barWidth - textResult.size.width) / 2,
                    size.height - 36.dp.toPx()
                )
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `rememberTextMeasurer()` | Measurer取得 |
| `measure()` | テキストサイズ計測 |
| `drawText()` | Canvas上にテキスト描画 |
| `TextLayoutResult.size` | 計測結果のサイズ |

- `rememberTextMeasurer`でComposable内でテキスト計測
- `measure()`で描画前にサイズ取得
- Canvas上のチャートラベル/バッジに活用
- `AnnotatedString`でスタイル付きテキストも計測可能

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [drawBehind/drawWithContent](https://zenn.dev/myougatheaxo/articles/android-compose-compose-draw-behind-2026)
- [テキストスタイル](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
