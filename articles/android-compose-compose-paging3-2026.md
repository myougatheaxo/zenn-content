---
title: "Compose Paging3完全ガイド — PagingSource/RemoteMediator/無限スクロール"
emoji: "📃"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

**Compose Paging3**（PagingSource、RemoteMediator、LazyPagingItems、無限スクロール）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("androidx.paging:paging-runtime-ktx:3.3.2")
    implementation("androidx.paging:paging-compose:3.3.2")
}
```

---

## PagingSource

```kotlin
class ArticlePagingSource(
    private val apiService: ApiService
) : PagingSource<Int, Article>() {

    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1
        return try {
            val response = apiService.getArticles(page = page, size = params.loadSize)
            LoadResult.Page(
                data = response.articles,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.articles.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}

// ViewModel
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val apiService: ApiService
) : ViewModel() {
    val articles: Flow<PagingData<Article>> = Pager(
        config = PagingConfig(pageSize = 20, prefetchDistance = 5),
        pagingSourceFactory = { ArticlePagingSource(apiService) }
    ).flow.cachedIn(viewModelScope)
}
```

---

## Compose UI

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn(Modifier.fillMaxSize()) {
        items(count = articles.itemCount, key = articles.itemKey { it.id }) { index ->
            articles[index]?.let { article ->
                ListItem(
                    headlineContent = { Text(article.title) },
                    supportingContent = { Text(article.summary, maxLines = 2) },
                    overlineContent = { Text(article.date) }
                )
                HorizontalDivider()
            }
        }

        // ローディング状態
        when (articles.loadState.append) {
            is LoadState.Loading -> item {
                Box(Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is LoadState.Error -> item {
                TextButton(onClick = { articles.retry() }, Modifier.fillMaxWidth()) {
                    Text("エラー。再試行")
                }
            }
            else -> {}
        }

        // 初回ローディング
        if (articles.loadState.refresh is LoadState.Loading) {
            item {
                Box(Modifier.fillParentMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
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
| `PagingSource` | データソース |
| `Pager` | ページングフロー生成 |
| `collectAsLazyPagingItems` | Compose連携 |
| `LoadState` | 読み込み状態 |

- `PagingSource`でAPIからの分割読み込みを定義
- `PagingConfig`でページサイズ・プリフェッチ距離を設定
- `cachedIn(viewModelScope)`で画面回転対応
- `LoadState`でローディング/エラー/完了を表示

---

8種類のAndroidアプリテンプレート（Paging3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose PagingCompose](https://zenn.dev/myougatheaxo/articles/android-compose-compose-paging-compose-2026)
- [Compose LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-column-2026)
- [Retrofit Paging](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-paging-2026)
