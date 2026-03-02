---
title: "オフラインファースト設計ガイド — Room + Retrofit同期"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**オフラインファースト**アーキテクチャ（ローカルDB優先、バックグラウンド同期、競合解決）を解説します。

---

## アーキテクチャ概要

```kotlin
// データフロー: UI ← Repository ← Room ← NetworkSync
// 1. UIはRoom(ローカルDB)からデータ取得
// 2. バックグラウンドでAPIからデータ取得→Room更新
// 3. オフラインでもRoom内のキャッシュで動作
```

---

## Repository パターン

```kotlin
class ArticleRepository(
    private val dao: ArticleDao,
    private val api: ArticleApi
) {
    // UIはこのFlowを監視（常にRoomから）
    fun getArticles(): Flow<List<Article>> = dao.getAllFlow()

    // ネットワーク同期
    suspend fun refresh(): Result<Unit> {
        return try {
            val articles = api.getArticles()
            dao.upsertAll(articles)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    // オフライン対応のCRUD
    suspend fun createArticle(article: Article) {
        // ローカルに即保存（pendingSync=true）
        dao.insert(article.copy(pendingSync = true))

        // バックグラウンドで同期
        try {
            api.createArticle(article)
            dao.update(article.copy(pendingSync = false))
        } catch (e: Exception) {
            // オフラインの場合、WorkManagerで後で同期
            enqueueSyncWork()
        }
    }
}
```

---

## Room DAO

```kotlin
@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles ORDER BY updatedAt DESC")
    fun getAllFlow(): Flow<List<Article>>

    @Upsert
    suspend fun upsertAll(articles: List<Article>)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(article: Article)

    @Update
    suspend fun update(article: Article)

    @Query("SELECT * FROM articles WHERE pendingSync = 1")
    suspend fun getPendingSync(): List<Article>
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val repository: ArticleRepository
) : ViewModel() {

    val articles = repository.getArticles()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing = _isRefreshing.asStateFlow()

    init { refresh() }

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            repository.refresh()
            _isRefreshing.value = false
        }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles by viewModel.articles.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() }
    ) {
        LazyColumn {
            items(articles, key = { it.id }) { article ->
                ArticleItem(
                    article = article,
                    showSyncIcon = article.pendingSync
                )
            }
        }
    }
}
```

---

## まとめ

- UIは常にRoom（ローカルDB）からデータ取得
- ネットワーク同期はバックグラウンドで実行
- `pendingSync`フラグでオフライン変更を追跡
- WorkManagerで未同期データのリトライ
- `Upsert`でAPIレスポンスをそのままDB更新
- `Flow`でDBの変更をリアクティブにUIへ反映

---

8種類のAndroidアプリテンプレート（オフライン対応設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room マイグレーション](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-2026)
- [WorkManager連携](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-2026)
- [ネットワーク監視](https://zenn.dev/myougatheaxo/articles/android-compose-network-status-2026)
