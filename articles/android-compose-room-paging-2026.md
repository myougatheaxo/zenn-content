---
title: "Room + Paging完全ガイド — PagingSource/RemoteMediator/オフラインキャッシュ"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

**Room + Paging**（PagingSource、RemoteMediator、オフラインキャッシュ、Compose LazyPaging）を解説します。

---

## Room PagingSource

```kotlin
@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles ORDER BY createdAt DESC")
    fun getArticlesPaging(): PagingSource<Int, Article>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(articles: List<Article>)

    @Query("DELETE FROM articles")
    suspend fun clearAll()
}
```

---

## RemoteMediator

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator @Inject constructor(
    private val api: ArticleApi,
    private val db: AppDatabase
) : RemoteMediator<Int, Article>() {

    override suspend fun load(loadType: LoadType, state: PagingState<Int, Article>): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> {
                val lastItem = state.lastItemOrNull() ?: return MediatorResult.Success(endOfPaginationReached = true)
                lastItem.page + 1
            }
        }

        return try {
            val response = api.getArticles(page = page, pageSize = state.config.pageSize)

            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    db.articleDao().clearAll()
                }
                db.articleDao().insertAll(response.map { it.copy(page = page) })
            }

            MediatorResult.Success(endOfPaginationReached = response.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

---

## Repository + ViewModel

```kotlin
class ArticleRepository @Inject constructor(
    private val db: AppDatabase,
    private val remoteMediator: ArticleRemoteMediator
) {
    @OptIn(ExperimentalPagingApi::class)
    fun getArticles(): Flow<PagingData<Article>> {
        return Pager(
            config = PagingConfig(pageSize = 20, prefetchDistance = 5),
            remoteMediator = remoteMediator,
            pagingSourceFactory = { db.articleDao().getArticlesPaging() }
        ).flow
    }
}

@HiltViewModel
class ArticleViewModel @Inject constructor(
    repository: ArticleRepository
) : ViewModel() {
    val articles = repository.getArticles().cachedIn(viewModelScope)
}

// Compose
@Composable
fun ArticleList(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn {
        items(articles.itemCount) { index ->
            articles[index]?.let { article ->
                ListItem(
                    headlineContent = { Text(article.title) },
                    supportingContent = { Text(article.summary) }
                )
            }
        }

        when (articles.loadState.append) {
            is LoadState.Loading -> item { CircularProgressIndicator(Modifier.fillMaxWidth().padding(16.dp)) }
            is LoadState.Error -> item { Text("読み込みエラー", Modifier.padding(16.dp)) }
            else -> {}
        }
    }
}
```

---

## まとめ

| コンポーネント | 役割 |
|---------------|------|
| `PagingSource` | ページ単位データ取得 |
| `RemoteMediator` | API→DB同期 |
| `Pager` | PagingData生成 |
| `collectAsLazyPagingItems` | Compose連携 |

- Room DAOが`PagingSource`を直接返却
- `RemoteMediator`でAPI→ローカルDBキャッシュ
- オフラインでもDBから表示可能
- `cachedIn(viewModelScope)`で再コンポーズ時の再取得防止

---

8種類のAndroidアプリテンプレート（Paging対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Paging3](https://zenn.dev/myougatheaxo/articles/android-compose-paging3-2026)
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
