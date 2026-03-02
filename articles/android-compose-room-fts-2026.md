---
title: "Room全文検索(FTS)完全ガイド — FTS4/日本語対応/検索ハイライト"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room FTS（全文検索）**（FTS4テーブル、日本語トークナイザー、検索ハイライト、Compose連携）を解説します。

---

## FTSエンティティ

```kotlin
@Entity(tableName = "articles")
data class Article(
    @PrimaryKey val id: Int,
    val title: String,
    val content: String,
    val category: String,
    val createdAt: Long = System.currentTimeMillis()
)

@Fts4(contentEntity = Article::class)
@Entity(tableName = "articles_fts")
data class ArticleFts(
    val title: String,
    val content: String
)
```

---

## DAO

```kotlin
@Dao
interface ArticleDao {
    @Query("""
        SELECT articles.* FROM articles
        JOIN articles_fts ON articles.rowid = articles_fts.rowid
        WHERE articles_fts MATCH :query
        ORDER BY rank
    """)
    fun search(query: String): Flow<List<Article>>

    @Query("""
        SELECT snippet(articles_fts, '<b>', '</b>', '...', 1, 30) as snippet
        FROM articles_fts
        WHERE articles_fts MATCH :query
    """)
    fun searchWithSnippet(query: String): Flow<List<SearchResult>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(article: Article)

    @Query("SELECT * FROM articles ORDER BY createdAt DESC")
    fun getAll(): Flow<List<Article>>
}

data class SearchResult(val snippet: String)
```

---

## Repository

```kotlin
class ArticleRepository @Inject constructor(
    private val dao: ArticleDao
) {
    fun search(query: String): Flow<List<Article>> {
        if (query.isBlank()) return dao.getAll()
        val ftsQuery = query.split(" ").joinToString(" ") { "$it*" }
        return dao.search(ftsQuery)
    }

    suspend fun insert(article: Article) = dao.insert(article)
}
```

---

## Compose検索画面

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    var query by remember { mutableStateOf("") }
    val results by viewModel.results.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize()) {
        OutlinedTextField(
            value = query,
            onValueChange = {
                query = it
                viewModel.search(it)
            },
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            placeholder = { Text("検索...") },
            leadingIcon = { Icon(Icons.Default.Search, null) },
            trailingIcon = {
                if (query.isNotEmpty()) {
                    IconButton(onClick = { query = ""; viewModel.search("") }) {
                        Icon(Icons.Default.Clear, "クリア")
                    }
                }
            },
            singleLine = true
        )

        LazyColumn {
            items(results, key = { it.id }) { article ->
                ListItem(
                    headlineContent = { HighlightedText(article.title, query) },
                    supportingContent = { Text(article.content.take(100)) }
                )
            }
        }
    }
}

@Composable
fun HighlightedText(text: String, query: String) {
    if (query.isBlank()) {
        Text(text)
        return
    }
    val annotated = buildAnnotatedString {
        var start = 0
        val lowerText = text.lowercase()
        val lowerQuery = query.lowercase()
        while (true) {
            val index = lowerText.indexOf(lowerQuery, start)
            if (index < 0) {
                append(text.substring(start))
                break
            }
            append(text.substring(start, index))
            withStyle(SpanStyle(background = Color.Yellow)) {
                append(text.substring(index, index + query.length))
            }
            start = index + query.length
        }
    }
    Text(annotated)
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| FTSテーブル | `@Fts4` |
| 全文検索 | `MATCH :query` |
| スニペット | `snippet()` |
| 前方一致 | `query*` |
| ハイライト | `AnnotatedString` |

- `@Fts4`でSQLiteの全文検索機能を活用
- `MATCH`演算子で高速テキスト検索
- `snippet()`で検索結果プレビュー生成
- `query*`で前方一致検索

---

8種類のAndroidアプリテンプレート（検索機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room CRUD](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-crud-2026)
- [ローカル検索](https://zenn.dev/myougatheaxo/articles/android-compose-local-search-2026)
- [SearchBar](https://zenn.dev/myougatheaxo/articles/android-compose-search-bar-2026)
