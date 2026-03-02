---
title: "Compose LazyStaggeredGrid完全ガイド — Pinterest風レイアウト/可変高さグリッド"
emoji: "🧱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**Compose LazyStaggeredGrid**（LazyVerticalStaggeredGrid、Pinterest風レイアウト、可変高さグリッド）を解説します。

---

## 基本LazyVerticalStaggeredGrid

```kotlin
@Composable
fun StaggeredGridDemo() {
    val items = remember {
        (1..30).map { it to (100..300).random() }
    }

    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Fixed(2),
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(8.dp),
        verticalItemSpacing = 8.dp,
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items) { (index, height) ->
            Card(Modifier.height(height.dp)) {
                Box(
                    Modifier
                        .fillMaxSize()
                        .background(Color(0xFF000000 + (index * 0x112233)),
                    contentAlignment = Alignment.Center
                ) {
                    Text("$index", color = Color.White, style = MaterialTheme.typography.titleLarge)
                }
            }
        }
    }
}
```

---

## 画像ギャラリー

```kotlin
@Composable
fun PhotoGallery(photos: List<Photo>) {
    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Adaptive(minSize = 150.dp),
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(4.dp),
        verticalItemSpacing = 4.dp,
        horizontalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        items(photos, key = { it.id }) { photo ->
            AsyncImage(
                model = photo.url,
                contentDescription = photo.description,
                modifier = Modifier
                    .fillMaxWidth()
                    .wrapContentHeight()
                    .clip(RoundedCornerShape(8.dp))
                    .clickable { /* 詳細へ */ },
                contentScale = ContentScale.FillWidth
            )
        }
    }
}
```

---

## Horizontal版

```kotlin
@Composable
fun HorizontalStaggeredDemo() {
    LazyHorizontalStaggeredGrid(
        rows = StaggeredGridCells.Fixed(3),
        modifier = Modifier.fillMaxWidth().height(300.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalItemSpacing = 8.dp,
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(20) { index ->
            Card(Modifier.width((80..200).random().dp).fillMaxHeight()) {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("Item $index")
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
| `LazyVerticalStaggeredGrid` | 縦スクロール千鳥格子 |
| `LazyHorizontalStaggeredGrid` | 横スクロール千鳥格子 |
| `StaggeredGridCells.Fixed` | 固定列数 |
| `StaggeredGridCells.Adaptive` | 最小幅指定 |

- Pinterest風の可変高さグリッドレイアウト
- `Adaptive`で画面サイズに応じた列数自動調整
- `wrapContentHeight()`で画像の自然な高さを維持
- `key`指定でリスト更新時のパフォーマンス最適化

---

8種類のAndroidアプリテンプレート（カスタムレイアウト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose LazyLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-layout-2026)
- [Compose ReorderableList](https://zenn.dev/myougatheaxo/articles/android-compose-compose-reorderable-list-2026)
- [Compose FlowColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-column-2026)
