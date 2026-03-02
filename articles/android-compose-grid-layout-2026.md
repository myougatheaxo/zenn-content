---
title: "LazyGridガイド — 固定列/適応型/スパン/スタッガード"
emoji: "🔲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

Composeの**LazyVerticalGrid/LazyHorizontalGrid**（固定列、適応型、スパン指定）を解説します。

---

## 固定列数

```kotlin
@Composable
fun FixedGrid(items: List<Product>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items, key = { it.id }) { product ->
            ProductCard(product)
        }
    }
}

@Composable
fun ProductCard(product: Product) {
    Card(Modifier.fillMaxWidth()) {
        Column {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = product.name,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(120.dp),
                contentScale = ContentScale.Crop
            )
            Column(Modifier.padding(8.dp)) {
                Text(product.name, style = MaterialTheme.typography.titleSmall)
                Text("¥${product.price}", style = MaterialTheme.typography.bodyMedium)
            }
        }
    }
}
```

---

## 適応型（Adaptive）

```kotlin
@Composable
fun AdaptiveGrid(items: List<Product>) {
    LazyVerticalGrid(
        // 最小幅150dpで自動的に列数を決定
        columns = GridCells.Adaptive(minSize = 150.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items, key = { it.id }) { product ->
            ProductCard(product)
        }
    }
}
```

---

## スパン指定

```kotlin
@Composable
fun SpanGrid(items: List<GridItem>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // ヘッダー（全幅）
        item(span = { GridItemSpan(maxLineSpan) }) {
            Text(
                "おすすめ商品",
                style = MaterialTheme.typography.titleMedium,
                modifier = Modifier.padding(vertical = 8.dp)
            )
        }

        // 特集アイテム（2列分）
        item(span = { GridItemSpan(2) }) {
            FeaturedCard(items.first())
        }

        // 通常アイテム（1列分）
        items(items.drop(1), key = { it.id }) { item ->
            SmallCard(item)
        }
    }
}
```

---

## ヘッダー付きセクション

```kotlin
@Composable
fun SectionedGrid(categories: Map<String, List<Product>>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        categories.forEach { (category, products) ->
            // セクションヘッダー
            item(span = { GridItemSpan(maxLineSpan) }) {
                Text(
                    category,
                    style = MaterialTheme.typography.titleMedium,
                    modifier = Modifier.padding(vertical = 8.dp)
                )
            }
            // セクション内アイテム
            items(products, key = { it.id }) { product ->
                ProductCard(product)
            }
        }
    }
}
```

---

## LazyHorizontalGrid

```kotlin
@Composable
fun HorizontalCategoryGrid(categories: List<Category>) {
    LazyHorizontalGrid(
        rows = GridCells.Fixed(2),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.height(200.dp)
    ) {
        items(categories, key = { it.id }) { category ->
            Card(
                Modifier.width(120.dp)
            ) {
                Column(
                    Modifier.padding(8.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Icon(category.icon, null, Modifier.size(32.dp))
                    Text(category.name, style = MaterialTheme.typography.bodySmall)
                }
            }
        }
    }
}
```

---

## まとめ

- `GridCells.Fixed(n)`: 固定n列
- `GridCells.Adaptive(minSize)`: 最小幅で自動列数
- `GridItemSpan(n)`: アイテムのスパン指定
- `maxLineSpan`: 全幅ヘッダーに使用
- `LazyHorizontalGrid`: 水平方向のグリッド
- `key`パラメータで効率的な再構成

---

8種類のAndroidアプリテンプレート（グリッドレイアウト設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyGrid完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazy-grid-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
- [レスポンシブレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-multi-screen-2026)
