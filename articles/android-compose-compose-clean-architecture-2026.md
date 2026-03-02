---
title: "Compose CleanArchitecture完全ガイド — レイヤー分離/UseCase/Repository/DI"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**Compose CleanArchitecture**（3層アーキテクチャ、UseCase、Repository、依存関係の方向）を解説します。

---

## レイヤー構成

```
UI層 (Presentation)          → ViewModel, Composable
 ↓
Domain層                     → UseCase, Entity, Repository Interface
 ↓
Data層                       → Repository実装, DataSource, API, DB
```

```kotlin
// Domain層: Entity
data class Article(val id: String, val title: String, val content: String, val date: Instant)

// Domain層: Repository Interface（依存逆転）
interface ArticleRepository {
    fun getArticles(): Flow<List<Article>>
    suspend fun getArticle(id: String): Article
    suspend fun saveArticle(article: Article)
}

// Domain層: UseCase
class GetArticlesUseCase @Inject constructor(
    private val repository: ArticleRepository
) {
    operator fun invoke(): Flow<List<Article>> =
        repository.getArticles().map { articles ->
            articles.sortedByDescending { it.date }
        }
}
```

---

## Data層

```kotlin
// Data層: Repository実装
class ArticleRepositoryImpl @Inject constructor(
    private val api: ArticleApi,
    private val dao: ArticleDao
) : ArticleRepository {

    override fun getArticles(): Flow<List<Article>> =
        dao.getAllArticles().map { entities ->
            entities.map { it.toDomain() }
        }

    override suspend fun getArticle(id: String): Article =
        dao.getArticle(id)?.toDomain() ?: api.getArticle(id).toDomain()

    override suspend fun saveArticle(article: Article) {
        dao.insert(article.toEntity())
    }
}

// Hilt DI
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds @Singleton
    abstract fun bindArticleRepository(impl: ArticleRepositoryImpl): ArticleRepository
}
```

---

## UI層

```kotlin
@HiltViewModel
class ArticleListViewModel @Inject constructor(
    private val getArticles: GetArticlesUseCase
) : ViewModel() {
    val articles: StateFlow<List<Article>> = getArticles()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}

@Composable
fun ArticleListScreen(viewModel: ArticleListViewModel = hiltViewModel()) {
    val articles by viewModel.articles.collectAsStateWithLifecycle()

    LazyColumn(Modifier.padding(16.dp)) {
        items(articles, key = { it.id }) { article ->
            ListItem(
                headlineContent = { Text(article.title) },
                supportingContent = { Text(article.content, maxLines = 2) }
            )
            HorizontalDivider()
        }
    }
}
```

---

## まとめ

| レイヤー | 責務 |
|----------|------|
| UI層 | 画面表示+ユーザー操作 |
| Domain層 | ビジネスロジック |
| Data層 | データアクセス実装 |

- Domain層は他層に依存しない（最も安定）
- Repository InterfaceはDomain層、実装はData層
- UseCaseで1つのビジネスロジックをカプセル化
- HiltでRepository InterfaceとImplを紐づけ

---

8種類のAndroidアプリテンプレート（Clean Architecture対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MVI](https://zenn.dev/myougatheaxo/articles/android-compose-compose-mvi-2026)
- [Hilt Module](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-module-2026)
- [Compose MultiModule](https://zenn.dev/myougatheaxo/articles/android-compose-compose-multi-module-2026)
