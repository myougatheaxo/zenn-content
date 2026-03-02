---
title: "Clean Architecture + Compose実践ガイド — レイヤー設計"
emoji: "🏛️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**Clean Architecture**のレイヤー設計（domain/data/presentation）をCompose+Hiltで解説します。

---

## レイヤー構成

```
app/
├── domain/          # ビジネスロジック（フレームワーク非依存）
│   ├── model/       # エンティティ
│   ├── repository/  # Repositoryインターフェース
│   └── usecase/     # ユースケース
├── data/            # データ層（実装）
│   ├── local/       # Room DAO
│   ├── remote/      # Retrofit API
│   └── repository/  # Repository実装
└── presentation/    # UI層
    ├── screen/      # Compose画面
    └── viewmodel/   # ViewModel
```

---

## Domain層

```kotlin
// model
data class Article(
    val id: String,
    val title: String,
    val content: String,
    val author: String,
    val createdAt: Long
)

// repository interface
interface ArticleRepository {
    fun getArticles(): Flow<List<Article>>
    suspend fun getArticleById(id: String): Article?
    suspend fun saveArticle(article: Article)
    suspend fun deleteArticle(id: String)
}

// usecase
class GetArticlesUseCase(private val repository: ArticleRepository) {
    operator fun invoke(): Flow<List<Article>> = repository.getArticles()
}

class SaveArticleUseCase(private val repository: ArticleRepository) {
    suspend operator fun invoke(article: Article) {
        require(article.title.isNotBlank()) { "Title is required" }
        repository.saveArticle(article)
    }
}
```

---

## Data層

```kotlin
// Entity (Room)
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey val id: String,
    val title: String,
    val content: String,
    val author: String,
    val createdAt: Long
) {
    fun toDomain() = Article(id, title, content, author, createdAt)

    companion object {
        fun fromDomain(article: Article) = ArticleEntity(
            article.id, article.title, article.content,
            article.author, article.createdAt
        )
    }
}

// Repository Implementation
class ArticleRepositoryImpl(
    private val dao: ArticleDao,
    private val api: ArticleApi
) : ArticleRepository {

    override fun getArticles(): Flow<List<Article>> =
        dao.getAllFlow().map { entities -> entities.map { it.toDomain() } }

    override suspend fun getArticleById(id: String): Article? =
        dao.getById(id)?.toDomain()

    override suspend fun saveArticle(article: Article) {
        dao.insert(ArticleEntity.fromDomain(article))
    }

    override suspend fun deleteArticle(id: String) = dao.deleteById(id)
}
```

---

## Presentation層

```kotlin
@HiltViewModel
class ArticleListViewModel @Inject constructor(
    getArticlesUseCase: GetArticlesUseCase,
    private val saveArticleUseCase: SaveArticleUseCase
) : ViewModel() {

    val articles = getArticlesUseCase()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun saveArticle(title: String, content: String) {
        viewModelScope.launch {
            saveArticleUseCase(
                Article(
                    id = UUID.randomUUID().toString(),
                    title = title,
                    content = content,
                    author = "me",
                    createdAt = System.currentTimeMillis()
                )
            )
        }
    }
}
```

---

## DI Module

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindArticleRepository(impl: ArticleRepositoryImpl): ArticleRepository
}

@Module
@InstallIn(ViewModelComponent::class)
object UseCaseModule {
    @Provides
    fun provideGetArticlesUseCase(repository: ArticleRepository) =
        GetArticlesUseCase(repository)

    @Provides
    fun provideSaveArticleUseCase(repository: ArticleRepository) =
        SaveArticleUseCase(repository)
}
```

---

## まとめ

- **Domain層**: フレームワーク非依存。Model + Repository IF + UseCase
- **Data層**: Room/Retrofit実装。Entity↔Domainの変換
- **Presentation層**: ViewModel + Compose画面
- UseCaseは`operator fun invoke()`で呼びやすく
- Hilt `@Binds`でRepository IF↔実装の紐付け
- テスト時はFake Repositoryに差し替え

---

8種類のAndroidアプリテンプレート（Clean Architecture設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt依存性注入](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
- [Repositoryパターン](https://zenn.dev/myougatheaxo/articles/kotlin-interface-patterns-2026)
- [ViewModel単体テスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-viewmodel-2026)
