---
title: "Room Multimap完全ガイド — Map返却/グループ集約/JOIN結果の変換"
emoji: "🗂️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Multimap**（Map返却、グループ集約クエリ、JOINのMap変換）を解説します。

---

## Map返却

```kotlin
@Dao
interface ArticleDao {
    // Map<Author, List<Article>>
    @Query("""
        SELECT * FROM Author
        JOIN Article ON Author.id = Article.authorId
    """)
    fun getAuthorArticles(): Flow<Map<Author, List<Article>>>

    // Map<String, List<Article>> カテゴリ別
    @MapColumn(columnName = "category")
    @Query("SELECT category, * FROM Article ORDER BY category, createdAt DESC")
    fun getArticlesByCategory(): Flow<Map<String, List<Article>>>
}
```

---

## 集約クエリ

```kotlin
@Dao
interface StatsDao {
    // 著者ごとの記事数
    @MapColumn(columnName = "name", keyColumn = "name")
    @MapColumn(columnName = "count", valueColumn = "count")
    @Query("""
        SELECT a.name, COUNT(ar.id) as count
        FROM Author a
        LEFT JOIN Article ar ON a.id = ar.authorId
        GROUP BY a.id
    """)
    fun getAuthorArticleCounts(): Flow<Map<String, Int>>
}
```

---

## Compose表示

```kotlin
@Composable
fun AuthorArticlesScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val authorArticles by viewModel.authorArticles.collectAsStateWithLifecycle()

    LazyColumn(Modifier.padding(16.dp)) {
        authorArticles.forEach { (author, articles) ->
            item {
                Text(
                    "${author.name} (${articles.size}記事)",
                    style = MaterialTheme.typography.titleMedium,
                    modifier = Modifier.padding(vertical = 8.dp)
                )
            }
            items(articles) { article ->
                ListItem(
                    headlineContent = { Text(article.title) },
                    supportingContent = { Text(article.summary) }
                )
            }
            item { HorizontalDivider(Modifier.padding(vertical = 8.dp)) }
        }
    }
}
```

---

## まとめ

| 機能 | 返却型 |
|------|--------|
| 1対多JOIN | `Map<Entity, List<Entity>>` |
| カラム→Map | `@MapColumn` |
| 集約 | `Map<String, Int>` |
| Flow対応 | `Flow<Map<...>>` |

- `Map`返却でJOIN結果を自動グループ化
- `@MapColumn`でカラム名とMapキーの対応
- 集約クエリの結果もMapで受け取り可能
- Room 2.4+で使用可能

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Relation](https://zenn.dev/myougatheaxo/articles/android-compose-room-relation-2026)
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room RawQuery](https://zenn.dev/myougatheaxo/articles/android-compose-room-rawquery-2026)
