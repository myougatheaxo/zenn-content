---
title: "Compose AppIndexing完全ガイド — Google検索連携/Firebase App Indexing/構造化データ"
emoji: "🔎"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "appindexing"]
published: true
---

## この記事で学べること

**Compose AppIndexing**（Firebase App Indexing、Google検索連携、構造化データ、ユーザーアクション記録）を解説します。

---

## App Indexing設定

```groovy
dependencies {
    implementation("com.google.firebase:firebase-appindexing:20.0.0")
}
```

---

## Indexable登録

```kotlin
class AppIndexingManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val firebaseAppIndex = FirebaseAppIndex.getInstance(context)

    fun indexArticle(article: Article) {
        val indexable = Indexables.digitalDocumentBuilder()
            .setName(article.title)
            .setDescription(article.summary)
            .setUrl("https://example.com/articles/${article.id}")
            .setMetadata(
                Indexable.Metadata.Builder()
                    .setWorksOffline(true)
            )
            .build()

        firebaseAppIndex.update(indexable)
    }

    fun indexRecipe(recipe: Recipe) {
        val indexable = Indexables.recipeBuilder()
            .setName(recipe.title)
            .setDescription(recipe.description)
            .setImage(recipe.imageUrl)
            .setUrl("https://example.com/recipes/${recipe.id}")
            .build()

        firebaseAppIndex.update(indexable)
    }

    fun removeIndex(url: String) {
        firebaseAppIndex.remove(url)
    }
}
```

---

## ユーザーアクション

```kotlin
@Composable
fun ArticleScreen(article: Article, indexingManager: AppIndexingManager) {
    // 閲覧時にインデックス更新
    LaunchedEffect(article.id) {
        indexingManager.indexArticle(article)

        val action = Actions.newView(article.title,
            "https://example.com/articles/${article.id}")
        FirebaseUserActions.getInstance(LocalContext.current).start(action)
    }

    DisposableEffect(article.id) {
        onDispose {
            val action = Actions.newView(article.title,
                "https://example.com/articles/${article.id}")
            FirebaseUserActions.getInstance().end(action)
        }
    }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text(article.title, style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))
        Text(article.content, style = MaterialTheme.typography.bodyLarge)
    }
}
```

---

## バッチインデックス

```kotlin
// 起動時に全コンテンツをインデックス
class IndexingWorker(
    context: Context, params: WorkerParameters
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val db = AppDatabase.getInstance(applicationContext)
        val articles = db.articleDao().getAll()

        val indexables = articles.map { article ->
            Indexables.digitalDocumentBuilder()
                .setName(article.title)
                .setUrl("https://example.com/articles/${article.id}")
                .build()
        }

        FirebaseAppIndex.getInstance(applicationContext).update(*indexables.toTypedArray())
        return Result.success()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FirebaseAppIndex` | インデックス管理 |
| `Indexables` | 構造化データ |
| `FirebaseUserActions` | アクション記録 |
| `Actions.newView` | 閲覧アクション |

- Firebase App IndexingでアプリコンテンツをGoogle検索に表示
- `Indexables`で構造化データ（記事、レシピ等）を定義
- `FirebaseUserActions`でユーザーの閲覧行動を記録
- WorkManagerでバックグラウンドバッチインデックス

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-deep-link-2026)
- [Compose AppLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-app-link-2026)
- [Compose FirebaseAnalytics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-analytics-2026)
