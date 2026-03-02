---
title: "Paging3 + Compose完全ガイド — LazyPagingItems実践"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

**Paging3**をComposeで使う方法（PagingSource、RemoteMediator、LazyPagingItems）を解説します。

---

## PagingSource

```kotlin
class ArticlePagingSource(
    private val api: ArticleApi
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
            val response = api.getArticles(page = page, size = params.loadSize)
            LoadResult.Page(
                data = response.items,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.items.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```

---

## ViewModel + Pager

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val api: ArticleApi
) : ViewModel() {

    val articles: Flow<PagingData<Article>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            prefetchDistance = 5,
            initialLoadSize = 40
        ),
        pagingSourceFactory = { ArticlePagingSource(api) }
    ).flow.cachedIn(viewModelScope)
}
```

---

## LazyPagingItems（Compose）

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = articles.itemCount,
            key = articles.itemKey { it.id }
        ) { index ->
            val article = articles[index]
            article?.let {
                ArticleCard(it)
            }
        }

        // ローディング状態
        when (articles.loadState.append) {
            is LoadState.Loading -> {
                item { LoadingItem() }
            }
            is LoadState.Error -> {
                item {
                    ErrorItem(
                        onRetry = { articles.retry() }
                    )
                }
            }
            else -> {}
        }
    }

    // 初期ローディング
    if (articles.loadState.refresh is LoadState.Loading) {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
    }
}

@Composable
fun LoadingItem() {
    Box(
        Modifier.fillMaxWidth().padding(16.dp),
        contentAlignment = Alignment.Center
    ) {
        CircularProgressIndicator(modifier = Modifier.size(24.dp))
    }
}

@Composable
fun ErrorItem(onRetry: () -> Unit) {
    Row(
        Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.Center
    ) {
        Text("読み込みエラー")
        Spacer(Modifier.width(8.dp))
        TextButton(onClick = onRetry) { Text("再試行") }
    }
}
```

---

## Pull-to-Refresh連携

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableArticleList(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()
    val isRefreshing = articles.loadState.refresh is LoadState.Loading

    val pullRefreshState = rememberPullToRefreshState()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { articles.refresh() },
        state = pullRefreshState
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(
                count = articles.itemCount,
                key = articles.itemKey { it.id }
            ) { index ->
                articles[index]?.let { ArticleCard(it) }
            }
        }
    }
}
```

---

## RemoteMediator（オフライン対応）

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val api: ArticleApi,
    private val db: AppDatabase
) : RemoteMediator<Int, ArticleEntity>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, ArticleEntity>
    ): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> {
                val lastItem = state.lastItemOrNull()
                    ?: return MediatorResult.Success(endOfPaginationReached = true)
                lastItem.nextPage
            }
        }

        return try {
            val response = api.getArticles(page = page, size = state.config.pageSize)
            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    db.articleDao().clearAll()
                }
                val entities = response.items.map { it.toEntity(nextPage = page + 1) }
                db.articleDao().insertAll(entities)
            }
            MediatorResult.Success(endOfPaginationReached = response.items.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

---

## まとめ

| コンポーネント | 役割 |
|--------------|------|
| `PagingSource` | データ取得ロジック |
| `Pager` | PagingConfig + SourceをFlowに変換 |
| `LazyPagingItems` | Compose用の状態管理 |
| `RemoteMediator` | ネットワーク+DB連携 |
| `LoadState` | 各段階の状態（Loading/Error/NotLoading） |

- `collectAsLazyPagingItems()`でCompose連携
- `loadState.append`で追加読み込みの状態管理
- `articles.retry()`でエラー時再試行
- `cachedIn(viewModelScope)`で画面回転対応

---

8種類のAndroidアプリテンプレート（ページング実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-tips-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
- [Pull-to-Refresh](https://zenn.dev/myougatheaxo/articles/android-compose-pull-refresh-2026)
