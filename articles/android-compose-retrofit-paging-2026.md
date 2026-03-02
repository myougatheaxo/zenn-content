---
title: "Retrofit Paging完全ガイド — Paging3+Retrofit/RemoteMediator/無限スクロール"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

**Retrofit Paging**（Paging3+Retrofit連携、PagingSource、RemoteMediator、無限スクロール）を解説します。

---

## PagingSource

```kotlin
class ArticlePagingSource(
    private val api: ArticleApi
) : PagingSource<Int, Article>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        return try {
            val page = params.key ?: 1
            val response = api.getArticles(page = page, perPage = params.loadSize)

            LoadResult.Page(
                data = response.data,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.data.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        return state.anchorPosition?.let { pos ->
            state.closestPageToPosition(pos)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(pos)?.nextKey?.minus(1)
        }
    }
}

interface ArticleApi {
    @GET("articles")
    suspend fun getArticles(
        @Query("page") page: Int,
        @Query("per_page") perPage: Int
    ): PaginatedResponse<Article>
}
```

---

## ViewModel + Compose

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val api: ArticleApi
) : ViewModel() {

    val articles: Flow<PagingData<Article>> = Pager(
        config = PagingConfig(pageSize = 20, prefetchDistance = 5),
        pagingSourceFactory = { ArticlePagingSource(api) }
    ).flow.cachedIn(viewModelScope)
}

@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn(Modifier.fillMaxSize()) {
        items(count = articles.itemCount, key = articles.itemKey { it.id }) { index ->
            articles[index]?.let { article ->
                ListItem(
                    headlineContent = { Text(article.title) },
                    supportingContent = { Text(article.summary) }
                )
            }
        }

        // ローディング表示
        when (articles.loadState.append) {
            is LoadState.Loading -> item {
                Box(Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is LoadState.Error -> item {
                TextButton(onClick = { articles.retry() }, Modifier.fillMaxWidth()) {
                    Text("再試行")
                }
            }
            else -> {}
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
    private val database: AppDatabase
) : RemoteMediator<Int, ArticleEntity>() {

    override suspend fun load(loadType: LoadType, state: PagingState<Int, ArticleEntity>): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> (state.lastItemOrNull()?.page ?: 0) + 1
        }

        return try {
            val response = api.getArticles(page, state.config.pageSize)
            database.withTransaction {
                if (loadType == LoadType.REFRESH) database.articleDao().clearAll()
                database.articleDao().insertAll(response.data.map { it.toEntity(page) })
            }
            MediatorResult.Success(endOfPaginationReached = response.data.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `PagingSource` | ページネーション |
| `Pager` | PagingData生成 |
| `collectAsLazyPagingItems` | Compose統合 |
| `RemoteMediator` | オフライン対応 |

- `PagingSource`でAPI→PagingData変換
- `cachedIn(viewModelScope)`でキャッシュ
- `LoadState`で読み込み状態別UI表示
- `RemoteMediator`でRoom+APIの同期

---

8種類のAndroidアプリテンプレート（ページネーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Paging](https://zenn.dev/myougatheaxo/articles/android-compose-room-paging-2026)
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Compose LazyLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-layout-2026)
