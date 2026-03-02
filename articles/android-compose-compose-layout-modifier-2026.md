---
title: "カスタムLayout/Modifier完全ガイド — Layout/Modifier.Node/intrinsicSize"
emoji: "📏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**カスタムLayout/Modifier**（Layout composable、Modifier.Node、intrinsicSize、SubcomposeLayout）を解説します。

---

## カスタムLayout

```kotlin
@Composable
fun FlowLayout(
    modifier: Modifier = Modifier,
    spacing: Dp = 8.dp,
    content: @Composable () -> Unit
) {
    Layout(content = content, modifier = modifier) { measurables, constraints ->
        val spacingPx = spacing.roundToPx()
        val placeables = measurables.map { it.measure(constraints.copy(minWidth = 0)) }

        var currentX = 0
        var currentY = 0
        var rowHeight = 0

        val positions = placeables.map { placeable ->
            if (currentX + placeable.width > constraints.maxWidth && currentX > 0) {
                currentX = 0
                currentY += rowHeight + spacingPx
                rowHeight = 0
            }
            val position = IntOffset(currentX, currentY)
            currentX += placeable.width + spacingPx
            rowHeight = maxOf(rowHeight, placeable.height)
            position
        }

        val height = currentY + rowHeight
        layout(constraints.maxWidth, height) {
            placeables.forEachIndexed { index, placeable ->
                placeable.place(positions[index])
            }
        }
    }
}

// 使用例: タグチップ
@Composable
fun TagChips(tags: List<String>) {
    FlowLayout(spacing = 8.dp) {
        tags.forEach { tag ->
            AssistChip(onClick = {}, label = { Text(tag) })
        }
    }
}
```

---

## カスタムModifier

```kotlin
fun Modifier.conditional(
    condition: Boolean,
    modifier: Modifier.() -> Modifier
): Modifier = if (condition) then(modifier()) else this

fun Modifier.shimmerEffect(): Modifier = composed {
    var size by remember { mutableStateOf(IntSize.Zero) }
    val transition = rememberInfiniteTransition(label = "shimmer")
    val startOffsetX by transition.animateFloat(
        initialValue = -2 * size.width.toFloat(),
        targetValue = 2 * size.width.toFloat(),
        animationSpec = infiniteRepeatable(tween(1000)),
        label = "shimmer"
    )

    background(
        brush = Brush.linearGradient(
            colors = listOf(Color(0xFFB8B5B5), Color(0xFF8F8B8B), Color(0xFFB8B5B5)),
            start = Offset(startOffsetX, 0f),
            end = Offset(startOffsetX + size.width, size.height.toFloat())
        )
    ).onGloballyPositioned { size = it.size }
}
```

---

## SubcomposeLayout

```kotlin
@Composable
fun MeasureTextLayout(text: String, content: @Composable (Int) -> Unit) {
    SubcomposeLayout { constraints ->
        val textPlaceable = subcompose("text") {
            Text(text, style = MaterialTheme.typography.bodyLarge)
        }.first().measure(constraints)

        val contentPlaceables = subcompose("content") {
            content(textPlaceable.width)
        }.map { it.measure(constraints) }

        val height = textPlaceable.height + contentPlaceables.sumOf { it.height }

        layout(constraints.maxWidth, height) {
            var y = 0
            textPlaceable.place(0, y)
            y += textPlaceable.height
            contentPlaceables.forEach { placeable ->
                placeable.place(0, y)
                y += placeable.height
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Layout` | カスタム配置 |
| `composed` | statefulなModifier |
| `SubcomposeLayout` | 動的サブコンポーズ |
| `Modifier.Node` | 高性能Modifier |

- `Layout`でFlowLayoutなどカスタム配置を実現
- `composed`でstateを持つModifierを作成
- `SubcomposeLayout`で先に計測してから配置
- パフォーマンス重視なら`Modifier.Node`を使用

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ConstraintLayout](https://zenn.dev/myougatheaxo/articles/android-compose-constraint-layout-2026)
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [再コンポーズ](https://zenn.dev/myougatheaxo/articles/android-compose-compose-recomposition-2026)
