---
title: "Room FTS4完全ガイド — 全文検索/Fts4Entity/MATCH/ハイライト"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room FTS4**（全文検索テーブル、@Fts4、MATCH構文、snippet/highlight）を解説します。

---

## FTS4エンティティ

```kotlin
@Entity
data class Article(
    @PrimaryKey(autoGenerate = true) val rowid: Long = 0,
    val title: String,
    val content: String,
    val author: String,
    val createdAt: Long = System.currentTimeMillis()
)

@Fts4(contentEntity = Article::class)
@Entity
data class ArticleFts(
    val title: String,
    val content: String,
    val author: String
)
```

---

## MATCH検索

```kotlin
@Dao
interface ArticleDao {
    @Query("""
        SELECT articles.* FROM articles
        JOIN ArticleFts ON articles.rowid = ArticleFts.rowid
        WHERE ArticleFts MATCH :query
        ORDER BY rank
    """)
    fun search(query: String): Flow<List<Article>>

    @Query("""
        SELECT articles.*, snippet(ArticleFts, '<b>', '</b>', '...', 1, 30) as snippet
        FROM articles
        JOIN ArticleFts ON articles.rowid = ArticleFts.rowid
        WHERE ArticleFts MATCH :query
    """)
    fun searchWithSnippet(query: String): Flow<List<SearchResult>>

    @Insert
    suspend fun insert(article: Article)
}

data class SearchResult(
    val rowid: Long,
    val title: String,
    val content: String,
    val author: String,
    val snippet: String
)
```

---

## Compose検索UI

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val articleDao: ArticleDao
) : ViewModel() {
    private val _query = MutableStateFlow("")
    val query = _query.asStateFlow()

    val results = _query
        .debounce(300)
        .filter { it.length >= 2 }
        .flatMapLatest { q -> articleDao.search("$q*") }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun updateQuery(q: String) { _query.value = q }
}

@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val results by viewModel.results.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = query,
            onValueChange = viewModel::updateQuery,
            placeholder = { Text("記事を検索...") },
            leadingIcon = { Icon(Icons.Default.Search, null) },
            modifier = Modifier.fillMaxWidth()
        )
        LazyColumn {
            items(results) { article ->
                ListItem(
                    headlineContent = { Text(article.title) },
                    supportingContent = { Text(article.content.take(100)) }
                )
            }
        }
    }
}
```

---

## まとめ

| 機能 | 説明 |
|------|------|
| `@Fts4` | FTS4仮想テーブル |
| `MATCH` | 全文検索クエリ |
| `snippet()` | 検索ハイライト |
| `contentEntity` | 元テーブル連携 |

- `@Fts4`でSQLiteの全文検索を活用
- `MATCH`構文でプレフィクス検索（`query*`）
- `snippet()`で検索結果のハイライト表示
- `debounce`でリアルタイム検索のパフォーマンス確保

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room Paging](https://zenn.dev/myougatheaxo/articles/android-compose-room-paging-2026)
- [Room Embedded](https://zenn.dev/myougatheaxo/articles/android-compose-room-embedded-2026)
