---
title: "Compose PagingCompose完全ガイド — LazyPagingItems/PlaceholderUI/リフレッシュ"
emoji: "♾️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

**Compose PagingCompose**（LazyPagingItems、Placeholder UI、Pull to Refresh連携、Grid対応）を解説します。

---

## LazyPagingItems活用

```kotlin
@Composable
fun PagingGridScreen(viewModel: ProductViewModel = hiltViewModel()) {
    val products = viewModel.products.collectAsLazyPagingItems()

    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        modifier = Modifier.padding(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(count = products.itemCount, key = products.itemKey { it.id }) { index ->
            products[index]?.let { product ->
                ProductCard(product)
            } ?: PlaceholderCard()
        }
    }
}

@Composable
fun PlaceholderCard() {
    ElevatedCard(Modifier.fillMaxWidth().height(200.dp)) {
        Box(Modifier.fillMaxSize().shimmerEffect()) // シマーアニメーション
    }
}

fun Modifier.shimmerEffect(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val alpha by transition.animateFloat(
        initialValue = 0.2f, targetValue = 0.9f,
        animationSpec = infiniteRepeatable(tween(1000), RepeatMode.Reverse),
        label = "shimmer"
    )
    background(Color.LightGray.copy(alpha = alpha))
}
```

---

## Pull to Refresh連携

```kotlin
@Composable
fun RefreshablePagingList(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()
    val isRefreshing = articles.loadState.refresh is LoadState.Loading

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { articles.refresh() }
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(count = articles.itemCount, key = articles.itemKey { it.id }) { index ->
                articles[index]?.let { article ->
                    ListItem(
                        headlineContent = { Text(article.title) },
                        supportingContent = { Text(article.summary) }
                    )
                    HorizontalDivider()
                }
            }

            // エラー表示
            if (articles.loadState.refresh is LoadState.Error) {
                item {
                    Column(Modifier.fillParentMaxSize(), horizontalAlignment = Alignment.CenterHorizontally,
                        verticalArrangement = Arrangement.Center) {
                        Text("データの読み込みに失敗しました")
                        Spacer(Modifier.height(8.dp))
                        Button(onClick = { articles.retry() }) { Text("再試行") }
                    }
                }
            }

            // 追加読み込みエラー
            if (articles.loadState.append is LoadState.Error) {
                item {
                    TextButton(onClick = { articles.retry() }, Modifier.fillMaxWidth()) {
                        Text("読み込みエラー。タップで再試行")
                    }
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
| `collectAsLazyPagingItems` | PagingDataをCompose化 |
| `itemKey` | アイテムキー指定 |
| `articles.refresh()` | 全データリフレッシュ |
| `articles.retry()` | エラーリトライ |

- `collectAsLazyPagingItems()`でPagingDataをLazyColumn対応に
- `itemKey`でアイテムのユニークキーを指定（リコンポジション最適化）
- `PullToRefreshBox`と`refresh()`で引っ張り更新
- `LoadState`でローディング/エラー/空を個別処理

---

8種類のAndroidアプリテンプレート（Paging3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Paging3](https://zenn.dev/myougatheaxo/articles/android-compose-compose-paging3-2026)
- [Compose PullRefresh](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pull-refresh-2026)
- [Compose LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-column-2026)
