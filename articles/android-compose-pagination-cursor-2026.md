---
title: "カーソルベースページネーション実装 — 無限スクロール"
emoji: "📜"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "pagination"]
published: true
---

## この記事で学べること

**カーソルベースページネーション**（無限スクロール、ロード状態管理、エラー再試行）を解説します。

---

## API設計

```kotlin
// カーソルベースAPI
interface ArticleApi {
    @GET("articles")
    suspend fun getArticles(
        @Query("cursor") cursor: String? = null,
        @Query("limit") limit: Int = 20
    ): PaginatedResponse<Article>
}

@Serializable
data class PaginatedResponse<T>(
    val data: List<T>,
    val nextCursor: String?,
    val hasMore: Boolean
)
```

---

## Repository

```kotlin
class ArticleRepository @Inject constructor(private val api: ArticleApi) {

    suspend fun getArticles(cursor: String? = null): PaginatedResponse<Article> {
        return api.getArticles(cursor = cursor)
    }
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class ArticleListViewModel @Inject constructor(
    private val repository: ArticleRepository
) : ViewModel() {

    private val _articles = MutableStateFlow<List<Article>>(emptyList())
    val articles: StateFlow<List<Article>> = _articles.asStateFlow()

    private val _loadState = MutableStateFlow<LoadState>(LoadState.Idle)
    val loadState: StateFlow<LoadState> = _loadState.asStateFlow()

    private var nextCursor: String? = null
    private var hasMore = true

    init { loadInitial() }

    fun loadInitial() {
        viewModelScope.launch {
            _loadState.value = LoadState.InitialLoading
            try {
                val response = repository.getArticles()
                _articles.value = response.data
                nextCursor = response.nextCursor
                hasMore = response.hasMore
                _loadState.value = LoadState.Idle
            } catch (e: Exception) {
                _loadState.value = LoadState.InitialError(e.message ?: "Error")
            }
        }
    }

    fun loadMore() {
        if (!hasMore || _loadState.value is LoadState.LoadingMore) return

        viewModelScope.launch {
            _loadState.value = LoadState.LoadingMore
            try {
                val response = repository.getArticles(cursor = nextCursor)
                _articles.update { it + response.data }
                nextCursor = response.nextCursor
                hasMore = response.hasMore
                _loadState.value = LoadState.Idle
            } catch (e: Exception) {
                _loadState.value = LoadState.LoadMoreError(e.message ?: "Error")
            }
        }
    }

    fun retry() {
        when (_loadState.value) {
            is LoadState.InitialError -> loadInitial()
            is LoadState.LoadMoreError -> loadMore()
            else -> {}
        }
    }
}

sealed interface LoadState {
    data object Idle : LoadState
    data object InitialLoading : LoadState
    data object LoadingMore : LoadState
    data class InitialError(val message: String) : LoadState
    data class LoadMoreError(val message: String) : LoadState
}
```

---

## Compose UI（無限スクロール）

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleListViewModel = hiltViewModel()) {
    val articles by viewModel.articles.collectAsStateWithLifecycle()
    val loadState by viewModel.loadState.collectAsStateWithLifecycle()
    val listState = rememberLazyListState()

    // 最後の要素が見えたら追加読み込み
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisibleIndex = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            lastVisibleIndex >= articles.size - 5 // 残り5件で先読み
        }
    }

    LaunchedEffect(shouldLoadMore) {
        if (shouldLoadMore) viewModel.loadMore()
    }

    when (loadState) {
        LoadState.InitialLoading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is LoadState.InitialError -> {
            ErrorScreen(
                message = (loadState as LoadState.InitialError).message,
                onRetry = { viewModel.retry() }
            )
        }
        else -> {
            LazyColumn(state = listState) {
                items(articles, key = { it.id }) { article ->
                    ArticleCard(article)
                }

                // 追加ローディング
                if (loadState is LoadState.LoadingMore) {
                    item {
                        Box(Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                            CircularProgressIndicator(Modifier.size(24.dp))
                        }
                    }
                }

                // 追加エラー
                if (loadState is LoadState.LoadMoreError) {
                    item {
                        TextButton(
                            onClick = { viewModel.retry() },
                            modifier = Modifier.fillMaxWidth()
                        ) {
                            Text("再読み込み")
                        }
                    }
                }
            }
        }
    }
}
```

---

## まとめ

- カーソルベースはオフセットより安全（挿入/削除時のズレなし）
- `derivedStateOf`で最後の要素検出→先読み
- `LoadState` sealed interfaceで初期/追加/エラー状態管理
- 残り5件で先読み開始（UX向上）
- Pull-to-Refreshは`loadInitial()`でリセット

---

8種類のAndroidアプリテンプレート（ページネーション実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Paging3+Compose](https://zenn.dev/myougatheaxo/articles/android-compose-paging3-compose-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-tips-2026)
- [Pull-to-Refresh](https://zenn.dev/myougatheaxo/articles/android-compose-pull-refresh-2026)
