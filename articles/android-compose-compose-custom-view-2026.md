---
title: "Compose CustomView完全ガイド — Layout/DrawScope/MeasurePolicy/カスタムコンポーネント"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "customview"]
published: true
---

## この記事で学べること

**Compose CustomView**（Layout、DrawScope、MeasurePolicy、カスタムレイアウト）を解説します。

---

## カスタムLayout

```kotlin
@Composable
fun FlowRow(
    modifier: Modifier = Modifier,
    spacing: Dp = 8.dp,
    content: @Composable () -> Unit
) {
    Layout(modifier = modifier, content = content) { measurables, constraints ->
        val spacingPx = spacing.roundToPx()
        val placeables = measurables.map { it.measure(constraints) }

        var x = 0
        var y = 0
        var rowHeight = 0

        val positions = placeables.map { placeable ->
            if (x + placeable.width > constraints.maxWidth) {
                x = 0
                y += rowHeight + spacingPx
                rowHeight = 0
            }
            val pos = IntOffset(x, y)
            x += placeable.width + spacingPx
            rowHeight = maxOf(rowHeight, placeable.height)
            pos
        }

        val totalHeight = y + rowHeight
        layout(constraints.maxWidth, totalHeight) {
            placeables.forEachIndexed { index, placeable ->
                placeable.place(positions[index])
            }
        }
    }
}

// 使用
@Composable
fun TagList(tags: List<String>) {
    FlowRow(Modifier.padding(16.dp)) {
        tags.forEach { tag ->
            AssistChip(onClick = {}, label = { Text(tag) })
        }
    }
}
```

---

## カスタム描画

```kotlin
@Composable
fun CircularProgress(progress: Float, modifier: Modifier = Modifier) {
    Canvas(modifier.size(100.dp)) {
        val strokeWidth = 8.dp.toPx()
        // 背景円
        drawArc(color = Color.LightGray, startAngle = 0f, sweepAngle = 360f,
            useCenter = false, style = Stroke(strokeWidth, cap = StrokeCap.Round))
        // プログレス円
        drawArc(color = Color(0xFF4CAF50), startAngle = -90f,
            sweepAngle = 360f * progress, useCenter = false,
            style = Stroke(strokeWidth, cap = StrokeCap.Round))
        // テキスト
        drawContext.canvas.nativeCanvas.apply {
            drawText("${(progress * 100).toInt()}%", size.width / 2,
                size.height / 2 + 12f, android.graphics.Paint().apply {
                    textAlign = android.graphics.Paint.Align.CENTER
                    textSize = 24f; color = android.graphics.Color.BLACK
                })
        }
    }
}
```

---

## SubcomposeLayout

```kotlin
@Composable
fun MeasuredText(text: String, content: @Composable (IntSize) -> Unit) {
    SubcomposeLayout { constraints ->
        val textPlaceable = subcompose("text") {
            Text(text)
        }.first().measure(constraints)

        val textSize = IntSize(textPlaceable.width, textPlaceable.height)
        val contentPlaceable = subcompose("content") {
            content(textSize)
        }.first().measure(constraints)

        layout(contentPlaceable.width, contentPlaceable.height) {
            contentPlaceable.place(0, 0)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Layout` | カスタムレイアウト |
| `Canvas` | カスタム描画 |
| `SubcomposeLayout` | サブコンポジション |
| `MeasurePolicy` | 計測ルール |

- `Layout`でmeasure→placeの流れでカスタム配置
- `Canvas`で自由な図形描画
- `SubcomposeLayout`で子の計測結果に基づく動的レイアウト
- パフォーマンスを考慮し、リコンポジションを最小限に

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
- [Compose IntrinsicSize](https://zenn.dev/myougatheaxo/articles/android-compose-compose-intrinsic-size-2026)
- [Compose GraphicsLayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-graphicsLayer-2026)
