---
title: "LazyLayout完全ガイド — カスタムLazy/LazyLayoutFactory/MeasurePolicy"
emoji: "🧱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**LazyLayout**（カスタムLazyコンポーネント、LazyLayoutItemProvider、MeasurePolicy）を解説します。

---

## LazyColumn/LazyRowの仕組み

```kotlin
@Composable
fun LazyColumnInternals() {
    // LazyColumnは内部的にLazyListを使用
    // 可視範囲のアイテムのみComposition
    val items = (1..1000).toList()

    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            count = items.size,
            key = { items[it] }  // 安定キーでリコンポジション最適化
        ) { index ->
            Card(Modifier.fillMaxWidth()) {
                Text(
                    "Item ${items[index]}",
                    Modifier.padding(16.dp)
                )
            }
        }
    }
}
```

---

## カスタムLazyレイアウト

```kotlin
@Composable
fun CustomFlowLayout(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        val placeables = measurables.map { it.measure(constraints.copy(minWidth = 0)) }
        var x = 0
        var y = 0
        var rowHeight = 0

        layout(constraints.maxWidth, constraints.maxHeight) {
            placeables.forEach { placeable ->
                if (x + placeable.width > constraints.maxWidth) {
                    x = 0
                    y += rowHeight
                    rowHeight = 0
                }
                placeable.placeRelative(x, y)
                x += placeable.width
                rowHeight = maxOf(rowHeight, placeable.height)
            }
        }
    }
}

@Composable
fun FlowLayoutDemo() {
    val tags = listOf("Kotlin", "Compose", "Android", "Room", "Hilt", "Retrofit", "Coroutines", "Flow")

    CustomFlowLayout(Modifier.padding(16.dp)) {
        tags.forEach { tag ->
            AssistChip(
                onClick = {},
                label = { Text(tag) },
                modifier = Modifier.padding(4.dp)
            )
        }
    }
}
```

---

## SubcomposeLayout

```kotlin
@Composable
fun MeasureFirstLayout(
    header: @Composable () -> Unit,
    content: @Composable (headerHeight: Int) -> Unit,
    modifier: Modifier = Modifier
) {
    SubcomposeLayout(modifier) { constraints ->
        val headerPlaceables = subcompose("header", header)
            .map { it.measure(constraints) }
        val headerHeight = headerPlaceables.maxOfOrNull { it.height } ?: 0

        val contentPlaceables = subcompose("content") { content(headerHeight) }
            .map { it.measure(constraints) }

        val totalHeight = headerHeight + (contentPlaceables.maxOfOrNull { it.height } ?: 0)

        layout(constraints.maxWidth, totalHeight) {
            headerPlaceables.forEach { it.placeRelative(0, 0) }
            contentPlaceables.forEach { it.placeRelative(0, headerHeight) }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LazyColumn/Row` | 標準Lazyリスト |
| `Layout` | カスタムレイアウト |
| `SubcomposeLayout` | 計測後にCompose |
| `key` | リコンポジション最適化 |

- `LazyColumn`は可視範囲のみComposition
- `Layout`でカスタムレイアウトロジック実装
- `SubcomposeLayout`で計測結果に基づくComposition
- `key`パラメータで差分更新を最適化

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyStaggeredGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-staggered-2026)
- [カスタムModifier](https://zenn.dev/myougatheaxo/articles/android-compose-compose-custom-modifier-2026)
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
