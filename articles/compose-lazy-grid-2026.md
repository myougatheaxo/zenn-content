---
title: "LazyVerticalGrid完全ガイド — Composeでグリッドレイアウトを実装"
emoji: "🔲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**LazyVerticalGrid** と **LazyHorizontalGrid** を使ったグリッドレイアウトの実装方法を解説します。

---

## 固定列数グリッド

```kotlin
@Composable
fun FixedGrid(items: List<Item>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(items, key = { it.id }) { item ->
            GridItem(item)
        }
    }
}

@Composable
fun GridItem(item: Item) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        shape = RoundedCornerShape(12.dp)
    ) {
        Column(Modifier.padding(12.dp)) {
            AsyncImage(
                model = item.imageUrl,
                contentDescription = null,
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(1f)
                    .clip(RoundedCornerShape(8.dp)),
                contentScale = ContentScale.Crop
            )
            Spacer(Modifier.height(8.dp))
            Text(item.title, style = MaterialTheme.typography.titleSmall)
            Text(item.subtitle, style = MaterialTheme.typography.bodySmall)
        }
    }
}
```

---

## アダプティブグリッド（最小幅指定）

```kotlin
LazyVerticalGrid(
    columns = GridCells.Adaptive(minSize = 150.dp),
    contentPadding = PaddingValues(16.dp),
    horizontalArrangement = Arrangement.spacedBy(8.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(products) { product ->
        ProductCard(product)
    }
}
```

画面幅に応じて列数が自動調整。タブレット対応に便利。

---

## ヘッダー付きグリッド

```kotlin
LazyVerticalGrid(
    columns = GridCells.Fixed(2),
    contentPadding = PaddingValues(16.dp),
    horizontalArrangement = Arrangement.spacedBy(12.dp),
    verticalArrangement = Arrangement.spacedBy(12.dp)
) {
    // ヘッダー（全幅）
    item(span = { GridItemSpan(maxLineSpan) }) {
        Text(
            "おすすめ商品",
            style = MaterialTheme.typography.headlineSmall,
            modifier = Modifier.padding(bottom = 8.dp)
        )
    }

    items(products) { product ->
        ProductCard(product)
    }

    // フッター（全幅）
    item(span = { GridItemSpan(maxLineSpan) }) {
        TextButton(
            onClick = { /* もっと見る */ },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("もっと見る")
        }
    }
}
```

`GridItemSpan(maxLineSpan)` で全幅アイテムを配置。

---

## カテゴリ別グリッド

```kotlin
@Composable
fun CategorizedGrid(categories: Map<String, List<Item>>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        categories.forEach { (category, items) ->
            // カテゴリヘッダー
            item(span = { GridItemSpan(maxLineSpan) }) {
                Text(
                    category,
                    style = MaterialTheme.typography.titleMedium,
                    modifier = Modifier.padding(vertical = 8.dp)
                )
            }

            items(items, key = { it.id }) { item ->
                GridItem(item)
            }
        }
    }
}
```

---

## スタガードグリッド

```kotlin
@Composable
fun StaggeredGrid(items: List<Item>) {
    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Fixed(2),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalItemSpacing = 12.dp
    ) {
        items(items, key = { it.id }) { item ->
            Card(shape = RoundedCornerShape(12.dp)) {
                Column {
                    AsyncImage(
                        model = item.imageUrl,
                        contentDescription = null,
                        modifier = Modifier.fillMaxWidth(),
                        contentScale = ContentScale.Crop
                    )
                    Text(
                        item.title,
                        modifier = Modifier.padding(12.dp),
                        style = MaterialTheme.typography.bodyMedium
                    )
                }
            }
        }
    }
}
```

Pinterest風のレイアウト。各アイテムの高さが異なっても自然に配置。

---

## パフォーマンス最適化

```kotlin
LazyVerticalGrid(
    columns = GridCells.Fixed(2),
    contentPadding = PaddingValues(16.dp),
    horizontalArrangement = Arrangement.spacedBy(12.dp),
    verticalArrangement = Arrangement.spacedBy(12.dp)
) {
    items(
        count = items.size,
        key = { items[it].id },  // key指定で効率的な再利用
        contentType = { items[it].type }  // contentType指定
    ) { index ->
        GridItem(items[index])
    }
}
```

- `key`でアイテムの同一性を指定
- `contentType`で同種アイテムのComposable再利用

---

## まとめ

- `GridCells.Fixed(n)`: 固定列数
- `GridCells.Adaptive(minSize)`: 画面幅で自動調整
- `GridItemSpan(maxLineSpan)`: ヘッダー・フッターの全幅表示
- `LazyVerticalStaggeredGrid`: Pinterest風レイアウト
- `key` + `contentType`でパフォーマンス最適化

---

8種類のAndroidアプリテンプレート（グリッドUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Adaptive Layout完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-adaptive-layout-2026)
