---
title: "Paging3 + Room連携ガイド — オフラインファーストページング"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

**Paging3 + Room**のオフラインファーストページング実装を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.paging:paging-runtime-ktx:3.3.5")
    implementation("androidx.paging:paging-compose:3.3.5")
    implementation("androidx.room:room-paging:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")
}
```

---

## Room Entity + DAO

```kotlin
@Entity
data class Article(
    @PrimaryKey val id: Long,
    val title: String,
    val content: String,
    val author: String,
    val createdAt: Long
)

@Dao
interface ArticleDao {
    @Query("SELECT * FROM article ORDER BY createdAt DESC")
    fun pagingSource(): PagingSource<Int, Article>

    @Upsert
    suspend fun upsertAll(articles: List<Article>)

    @Query("DELETE FROM article")
    suspend fun clearAll()
}
```

---

## RemoteMediator

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val api: ArticleApi,
    private val db: AppDatabase
) : RemoteMediator<Int, Article>() {

    private val dao = db.articleDao()

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, Article>
    ): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> {
                val lastItem = state.lastItemOrNull()
                    ?: return MediatorResult.Success(endOfPaginationReached = true)
                (lastItem.id / 20 + 1).toInt()
            }
        }

        return try {
            val response = api.getArticles(page = page, pageSize = 20)

            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    dao.clearAll()
                }
                dao.upsertAll(response.articles)
            }

            MediatorResult.Success(endOfPaginationReached = response.articles.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

---

## Repository

```kotlin
class ArticleRepository(
    private val api: ArticleApi,
    private val db: AppDatabase
) {
    @OptIn(ExperimentalPagingApi::class)
    fun getArticlesPager(): Flow<PagingData<Article>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                prefetchDistance = 5,
                initialLoadSize = 40
            ),
            remoteMediator = ArticleRemoteMediator(api, db),
            pagingSourceFactory = { db.articleDao().pagingSource() }
        ).flow
    }
}
```

---

## ViewModel

```kotlin
class ArticleViewModel(
    private val repository: ArticleRepository
) : ViewModel() {

    val articles: Flow<PagingData<Article>> =
        repository.getArticlesPager().cachedIn(viewModelScope)
}
```

---

## Compose画面

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = viewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn(Modifier.fillMaxSize()) {
        items(
            count = articles.itemCount,
            key = articles.itemKey { it.id }
        ) { index ->
            val article = articles[index]
            article?.let {
                ListItem(
                    headlineContent = { Text(it.title) },
                    supportingContent = { Text(it.author) }
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
                TextButton(
                    onClick = { articles.retry() },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text("再試行")
                }
            }
            else -> {}
        }
    }

    // Pull-to-Refreshとの連携
    PullToRefreshBox(
        isRefreshing = articles.loadState.refresh is LoadState.Loading,
        onRefresh = { articles.refresh() }
    ) {
        // LazyColumn...
    }
}
```

---

## まとめ

- `Room.pagingSource()`でローカルDBからページング
- `RemoteMediator`でAPIデータをDBに同期
- `Pager`で`PagingConfig` + `RemoteMediator` + `PagingSource`を統合
- `collectAsLazyPagingItems()`でCompose連携
- `loadState`でローディング/エラー状態表示
- `cachedIn(viewModelScope)`でキャッシュ

---

8種類のAndroidアプリテンプレート（Paging3対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Paging3完全ガイド](https://zenn.dev/myougatheaxo/articles/android-paging3-guide-2026)
- [Room Databaseガイド](https://zenn.dev/myougatheaxo/articles/android-room-database-2026)
- [Retrofit通信ガイド](https://zenn.dev/myougatheaxo/articles/android-retrofit-network-2026)
