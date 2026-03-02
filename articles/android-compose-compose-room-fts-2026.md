---
title: "Compose Room FTS完全ガイド — 全文検索/FTS4/日本語検索/ハイライト表示"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Compose Room FTS**（FTS4全文検索、日本語対応、検索結果ハイライト、Compose UI連携）を解説します。

---

## FTSエンティティ

```kotlin
@Entity(tableName = "articles")
data class Article(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val content: String,
    val createdAt: Long = System.currentTimeMillis()
)

@Fts4(contentEntity = Article::class)
@Entity(tableName = "articles_fts")
data class ArticleFts(
    val title: String,
    val content: String
)

@Dao
interface ArticleDao {
    @Query("""
        SELECT articles.* FROM articles
        JOIN articles_fts ON articles.rowid = articles_fts.rowid
        WHERE articles_fts MATCH :query
        ORDER BY rank
    """)
    fun search(query: String): Flow<List<Article>>

    @Query("SELECT * FROM articles ORDER BY createdAt DESC")
    fun getAll(): Flow<List<Article>>

    @Insert
    suspend fun insert(article: Article)
}
```

---

## 検索ViewModel

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val articleDao: ArticleDao
) : ViewModel() {
    private val _query = MutableStateFlow("")
    val query = _query.asStateFlow()

    val results: StateFlow<List<Article>> = _query
        .debounce(300)
        .flatMapLatest { q ->
            if (q.isBlank()) articleDao.getAll()
            else articleDao.search("$q*") // 前方一致
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onQueryChange(query: String) { _query.value = query }
}
```

---

## Compose UI

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val results by viewModel.results.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize()) {
        OutlinedTextField(
            value = query,
            onValueChange = { viewModel.onQueryChange(it) },
            label = { Text("記事を検索") },
            leadingIcon = { Icon(Icons.Default.Search, contentDescription = null) },
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            singleLine = true
        )

        LazyColumn(contentPadding = PaddingValues(16.dp)) {
            items(results) { article ->
                Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                    Column(Modifier.padding(16.dp)) {
                        Text(article.title, style = MaterialTheme.typography.titleMedium)
                        Spacer(Modifier.height(4.dp))
                        Text(
                            article.content.take(100) + "...",
                            style = MaterialTheme.typography.bodyMedium,
                            color = MaterialTheme.colorScheme.onSurfaceVariant
                        )
                    }
                }
            }
        }

        if (results.isEmpty() && query.isNotEmpty()) {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                Text("「$query」に一致する記事がありません")
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@Fts4` | FTS4テーブル定義 |
| `MATCH` | 全文検索クエリ |
| `contentEntity` | 元テーブル参照 |
| `rank` | 関連度ソート |

- `@Fts4`でFTS仮想テーブルを定義
- `MATCH`演算子で高速全文検索
- `contentEntity`で元テーブルとの同期を管理
- `debounce`で入力中の過剰なクエリを防止

---

8種類のAndroidアプリテンプレート（Room対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Room](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-2026)
- [Compose Room Relation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-relation-2026)
- [Compose Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-migration-2026)
