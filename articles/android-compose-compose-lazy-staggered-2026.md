---
title: "StaggeredGrid完全ガイド — LazyVerticalStaggeredGrid/Pinterest風/Adaptive"
emoji: "🧱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**StaggeredGrid**（LazyVerticalStaggeredGrid、Pinterest風レイアウト、StaggeredGridCells.Adaptive）を解説します。

---

## 基本StaggeredGrid

```kotlin
@Composable
fun StaggeredGridExample() {
    val items = remember {
        List(30) { index ->
            Pair("Item $index", (100..300).random().dp)
        }
    }

    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Fixed(2),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalItemSpacing = 8.dp
    ) {
        items(items.size) { index ->
            val (title, height) = items[index]
            Card(
                modifier = Modifier.fillMaxWidth().height(height),
                shape = RoundedCornerShape(12.dp)
            ) {
                Box(
                    Modifier
                        .fillMaxSize()
                        .background(
                            Color(
                                red = (100..200).random(),
                                green = (100..200).random(),
                                blue = (100..200).random()
                            )
                        ),
                    contentAlignment = Alignment.Center
                ) {
                    Text(title, color = Color.White, fontWeight = FontWeight.Bold)
                }
            }
        }
    }
}
```

---

## Pinterest風画像グリッド

```kotlin
@Composable
fun PinterestGrid(images: List<ImageItem>) {
    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Adaptive(150.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalItemSpacing = 8.dp
    ) {
        items(images, key = { it.id }) { image ->
            Card(shape = RoundedCornerShape(12.dp)) {
                Column {
                    AsyncImage(
                        model = image.url,
                        contentDescription = null,
                        modifier = Modifier
                            .fillMaxWidth()
                            .aspectRatio(image.aspectRatio),
                        contentScale = ContentScale.Crop
                    )
                    Text(
                        image.title,
                        modifier = Modifier.padding(8.dp),
                        maxLines = 2,
                        overflow = TextOverflow.Ellipsis
                    )
                }
            }
        }
    }
}

data class ImageItem(
    val id: Long,
    val url: String,
    val title: String,
    val aspectRatio: Float
)
```

---

## Horizontal StaggeredGrid

```kotlin
@Composable
fun HorizontalStaggeredExample() {
    LazyHorizontalStaggeredGrid(
        rows = StaggeredGridCells.Fixed(3),
        contentPadding = PaddingValues(16.dp),
        horizontalItemSpacing = 8.dp,
        verticalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.height(300.dp)
    ) {
        items(20) { index ->
            Card(
                modifier = Modifier.width((80..200).random().dp),
                shape = RoundedCornerShape(8.dp)
            ) {
                Text("Tag $index", Modifier.padding(12.dp))
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LazyVerticalStaggeredGrid` | 縦方向不揃いグリッド |
| `LazyHorizontalStaggeredGrid` | 横方向不揃いグリッド |
| `StaggeredGridCells.Fixed` | 固定列数 |
| `StaggeredGridCells.Adaptive` | 適応列数 |

- Pinterest風の不揃い高さのグリッドレイアウト
- `StaggeredGridCells.Adaptive`で画面幅に自動適応
- `aspectRatio`で画像の比率を維持
- LazyColumnと同様のパフォーマンス最適化

---

8種類のAndroidアプリテンプレート（グリッドレイアウト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-grid-2026)
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
- [FlowRow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-row-2026)
