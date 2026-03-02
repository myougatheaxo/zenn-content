---
title: "Paging 3完全ガイド — 大量データを効率的にロードする"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

数百〜数万件のデータを**ページごとに自動で読み込む** Paging 3ライブラリの使い方を解説します。LazyColumnとの統合パターンも紹介。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.paging:paging-runtime-ktx:3.3.0")
    implementation("androidx.paging:paging-compose:3.3.0")
}
```

---

## PagingSource

```kotlin
class ArticlePagingSource(
    private val api: ArticleApi
) : PagingSource<Int, Article>() {

    override suspend fun load(
        params: LoadParams<Int>
    ): LoadResult<Int, Article> {
        val page = params.key ?: 1

        return try {
            val response = api.getArticles(
                page = page,
                pageSize = params.loadSize
            )

            LoadResult.Page(
                data = response.articles,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.articles.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }
}
```

---

## ViewModel

```kotlin
class ArticleViewModel(private val api: ArticleApi) : ViewModel() {
    val articles: Flow<PagingData<Article>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            prefetchDistance = 5,
            enablePlaceholders = false
        )
    ) {
        ArticlePagingSource(api)
    }.flow.cachedIn(viewModelScope)
}
```

| パラメータ | 説明 |
|-----------|------|
| `pageSize` | 1ページあたりの件数 |
| `prefetchDistance` | 何件前から次ページを先読み |
| `enablePlaceholders` | プレースホルダー表示 |

---

## ComposeでのUI

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = articles.itemCount,
            key = articles.itemKey { it.id }
        ) { index ->
            val article = articles[index]
            if (article != null) {
                ArticleItem(article)
            }
        }

        // ローディングインジケーター
        when (articles.loadState.append) {
            is LoadState.Loading -> {
                item {
                    CircularProgressIndicator(
                        Modifier.fillMaxWidth().padding(16.dp)
                    )
                }
            }
            is LoadState.Error -> {
                item {
                    Button(onClick = { articles.retry() }) {
                        Text("再試行")
                    }
                }
            }
            else -> {}
        }
    }
}
```

---

## Pull-to-Refresh

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableList(viewModel: ArticleViewModel) {
    val articles = viewModel.articles.collectAsLazyPagingItems()
    val isRefreshing = articles.loadState.refresh is LoadState.Loading

    val pullToRefreshState = rememberPullToRefreshState()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { articles.refresh() },
        state = pullToRefreshState
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(
                count = articles.itemCount,
                key = articles.itemKey { it.id }
            ) { index ->
                articles[index]?.let { ArticleItem(it) }
            }
        }
    }
}
```

---

## Room + Paging（オフライン対応）

```kotlin
@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles ORDER BY createdAt DESC")
    fun pagingSource(): PagingSource<Int, ArticleEntity>
}
```

```kotlin
val articles = Pager(
    config = PagingConfig(pageSize = 20)
) {
    dao.pagingSource()
}.flow.cachedIn(viewModelScope)
```

RoomのDAOが直接`PagingSource`を返せるため、**オフライン対応が簡単**です。

---

## まとめ

- `PagingSource`でデータ取得ロジックを定義
- `Pager` + `PagingConfig`でページング設定
- `collectAsLazyPagingItems()`でComposeと統合
- `loadState`でローディング/エラー状態を表示
- Room連携でオフライン対応も可能

---

8種類のAndroidアプリテンプレート（効率的なデータロード設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
- [Room Migration完全ガイド](https://zenn.dev/myougatheaxo/articles/room-migration-guide-2026)
