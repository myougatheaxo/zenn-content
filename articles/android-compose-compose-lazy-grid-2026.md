---
title: "LazyVerticalGrid完全ガイド — グリッド表示/スパン/スティッキーヘッダー"
emoji: "🔲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**LazyVerticalGrid**（グリッド表示、固定/適応カラム、スパンサイズ、スティッキーヘッダー）を解説します。

---

## 基本グリッド

```kotlin
@Composable
fun PhotoGrid(photos: List<Photo>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(4.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        items(photos, key = { it.id }) { photo ->
            AsyncImage(
                model = photo.url,
                contentDescription = photo.title,
                contentScale = ContentScale.Crop,
                modifier = Modifier.aspectRatio(1f).clip(RoundedCornerShape(4.dp))
            )
        }
    }
}
```

---

## 適応カラム + スパン

```kotlin
@Composable
fun AdaptiveGrid(items: List<Item>) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 150.dp),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // フルスパンヘッダー
        item(span = { GridItemSpan(maxLineSpan) }) {
            Text("おすすめ", style = MaterialTheme.typography.headlineSmall, modifier = Modifier.padding(vertical = 8.dp))
        }

        // 通常アイテム
        items(items) { item ->
            Card(Modifier.fillMaxWidth()) {
                Column(Modifier.padding(12.dp)) {
                    AsyncImage(model = item.imageUrl, contentDescription = null, modifier = Modifier.fillMaxWidth().height(120.dp), contentScale = ContentScale.Crop)
                    Text(item.title, style = MaterialTheme.typography.titleSmall)
                    Text("¥${item.price}", color = MaterialTheme.colorScheme.primary)
                }
            }
        }
    }
}
```

---

## LazyVerticalStaggeredGrid

```kotlin
@Composable
fun StaggeredPhotoGrid(photos: List<Photo>) {
    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Fixed(2),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalItemSpacing = 8.dp
    ) {
        items(photos) { photo ->
            Card {
                Column {
                    AsyncImage(
                        model = photo.url,
                        contentDescription = null,
                        modifier = Modifier.fillMaxWidth().height((100 + (photo.id % 3) * 50).dp),
                        contentScale = ContentScale.Crop
                    )
                    Text(photo.title, Modifier.padding(8.dp))
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GridCells.Fixed` | 固定カラム数 |
| `GridCells.Adaptive` | 最小幅で自動 |
| `GridItemSpan` | スパンサイズ |
| `StaggeredGrid` | 高さ不揃い |

- `LazyVerticalGrid`でグリッド表示
- `GridCells.Adaptive`で画面幅に応じた列数
- `GridItemSpan(maxLineSpan)`でフルスパンヘッダー
- `LazyVerticalStaggeredGrid`でPinterest風レイアウト

---

8種類のAndroidアプリテンプレート（グリッドUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyLayout](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-layout-2026)
- [FlowRow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-row-2026)
- [Coil画像](https://zenn.dev/myougatheaxo/articles/android-compose-coil-image-2026)
