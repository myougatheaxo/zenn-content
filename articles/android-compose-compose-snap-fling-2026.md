---
title: "SnapFling完全ガイド — SnapFlingBehavior/ページスナップ/アイテムスナップ"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "scroll"]
published: true
---

## この記事で学べること

**SnapFling**（SnapFlingBehavior、ページスナップ、LazyRowスナップ）を解説します。

---

## SnapFlingBehavior

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun SnapFlingExample() {
    val listState = rememberLazyListState()
    val flingBehavior = rememberSnapFlingBehavior(lazyListState = listState)

    LazyRow(
        state = listState,
        flingBehavior = flingBehavior,
        contentPadding = PaddingValues(horizontal = 32.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        items(10) { index ->
            Card(
                modifier = Modifier.width(300.dp).height(200.dp),
                shape = RoundedCornerShape(16.dp)
            ) {
                Box(
                    Modifier.fillMaxSize().background(
                        Color(
                            red = (100..200).random(),
                            green = (100..200).random(),
                            blue = (100..200).random()
                        )
                    ),
                    contentAlignment = Alignment.Center
                ) {
                    Text("Card $index", color = Color.White, fontSize = 24.sp)
                }
            }
        }
    }
}
```

---

## カルーセル with スナップ

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun CarouselWithSnap(items: List<CarouselItem>) {
    val listState = rememberLazyListState()
    val flingBehavior = rememberSnapFlingBehavior(lazyListState = listState)

    Column {
        LazyRow(
            state = listState,
            flingBehavior = flingBehavior,
            contentPadding = PaddingValues(horizontal = 24.dp),
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            itemsIndexed(items) { index, item ->
                val currentIndex by remember {
                    derivedStateOf { listState.firstVisibleItemIndex }
                }
                Card(
                    modifier = Modifier
                        .width(280.dp)
                        .graphicsLayer {
                            val scale = if (index == currentIndex) 1f else 0.9f
                            scaleX = scale
                            scaleY = scale
                        },
                    shape = RoundedCornerShape(16.dp)
                ) {
                    AsyncImage(
                        model = item.imageUrl,
                        contentDescription = null,
                        modifier = Modifier.fillMaxWidth().height(180.dp),
                        contentScale = ContentScale.Crop
                    )
                    Text(item.title, Modifier.padding(12.dp))
                }
            }
        }

        // インジケーター
        Row(
            Modifier.fillMaxWidth().padding(8.dp),
            horizontalArrangement = Arrangement.Center
        ) {
            repeat(items.size) { index ->
                Box(
                    Modifier
                        .padding(2.dp)
                        .size(if (listState.firstVisibleItemIndex == index) 10.dp else 6.dp)
                        .clip(CircleShape)
                        .background(
                            if (listState.firstVisibleItemIndex == index)
                                MaterialTheme.colorScheme.primary
                            else MaterialTheme.colorScheme.outline
                        )
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `rememberSnapFlingBehavior` | スナップ動作 |
| `flingBehavior` | LazyRow/Columnに適用 |
| `contentPadding` | 端のピーク表示 |
| `graphicsLayer` | スケールエフェクト |

- `rememberSnapFlingBehavior`でアイテム単位のスナップ
- `contentPadding`で隣のカードをチラ見せ
- `graphicsLayer`で現在アイテムを拡大表示
- カルーセルUIに最適

---

8種類のAndroidアプリテンプレート（カルーセル対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Pager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pager-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
