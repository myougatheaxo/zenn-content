---
title: "カスタムレイアウト実装 — Compose Layoutで自由なUI配置を作る"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

Composeの**カスタムLayout**とMeasure/Placeの仕組みを解説します。

---

## Layout の基本

```kotlin
@Composable
fun SimpleCustomLayout(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        // 1. 子要素を測定
        val placeables = measurables.map { it.measure(constraints) }

        // 2. レイアウトサイズを決定
        val width = constraints.maxWidth
        val height = placeables.sumOf { it.height }

        layout(width, height) {
            // 3. 子要素を配置
            var yPosition = 0
            placeables.forEach { placeable ->
                placeable.placeRelative(0, yPosition)
                yPosition += placeable.height
            }
        }
    }
}
```

---

## 横並びレイアウト（均等配置）

```kotlin
@Composable
fun EvenRow(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(content = content, modifier = modifier) { measurables, constraints ->
        val itemWidth = constraints.maxWidth / measurables.size
        val itemConstraints = constraints.copy(
            minWidth = itemWidth,
            maxWidth = itemWidth
        )
        val placeables = measurables.map { it.measure(itemConstraints) }
        val height = placeables.maxOf { it.height }

        layout(constraints.maxWidth, height) {
            placeables.forEachIndexed { index, placeable ->
                placeable.placeRelative(index * itemWidth, 0)
            }
        }
    }
}

// 使用例
EvenRow(Modifier.fillMaxWidth()) {
    Text("A", textAlign = TextAlign.Center)
    Text("B", textAlign = TextAlign.Center)
    Text("C", textAlign = TextAlign.Center)
}
```

---

## オーバーラップレイアウト

```kotlin
@Composable
fun OverlappingLayout(
    overlapFactor: Float = 0.3f,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(content = content, modifier = modifier) { measurables, constraints ->
        val placeables = measurables.map { it.measure(constraints) }
        val itemWidth = placeables.firstOrNull()?.width ?: 0

        val totalWidth = if (placeables.isEmpty()) 0
            else itemWidth + (placeables.size - 1) * (itemWidth * (1 - overlapFactor)).toInt()
        val height = placeables.maxOfOrNull { it.height } ?: 0

        layout(totalWidth, height) {
            placeables.forEachIndexed { index, placeable ->
                val x = (index * itemWidth * (1 - overlapFactor)).toInt()
                placeable.placeRelative(x, 0)
            }
        }
    }
}

// アバターの重なり表示
OverlappingLayout(overlapFactor = 0.25f) {
    repeat(5) {
        AsyncImage(
            model = avatarUrls[it],
            contentDescription = null,
            modifier = Modifier.size(40.dp).clip(CircleShape)
        )
    }
}
```

---

## Modifier.layout

```kotlin
// 個別要素のレイアウト修正
fun Modifier.firstBaselineToTop(firstBaselineToTop: Dp) = layout { measurable, constraints ->
    val placeable = measurable.measure(constraints)
    val firstBaseline = placeable[FirstBaseline]
    val y = firstBaselineToTop.roundToPx() - firstBaseline

    layout(placeable.width, placeable.height + y) {
        placeable.placeRelative(0, y)
    }
}

// 使用例
Text(
    "ベースライン調整",
    Modifier.firstBaselineToTop(32.dp)
)
```

---

## IntrinsicSize

```kotlin
@Composable
fun IntrinsicExample() {
    // 子要素の最大固有幅に揃える
    Row(Modifier.height(IntrinsicSize.Min)) {
        Text("短い", Modifier.weight(1f))
        HorizontalDivider(
            Modifier.fillMaxHeight().width(1.dp),
            color = Color.Gray
        )
        Text(
            "これは長いテキストです。自動的に折り返されます。",
            Modifier.weight(1f)
        )
    }
}
```

---

## まとめ

- `Layout`コンポーザブルで完全カスタムレイアウト
- `measurables.map { it.measure() }`で子要素を測定
- `layout(width, height) { placeable.placeRelative() }`で配置
- `Modifier.layout`で個別要素のレイアウト修正
- `IntrinsicSize`で子要素サイズの事前測定
- 標準のRow/Column/Boxで足りない時だけ使う

---

8種類のAndroidアプリテンプレート（カスタムUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Modifier完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-modifier-guide-2026)
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [ConstraintLayoutガイド](https://zenn.dev/myougatheaxo/articles/compose-constraint-layout-2026)
