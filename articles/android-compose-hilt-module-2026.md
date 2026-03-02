---
title: "Hilt Module完全ガイド — @Module/@Provides/@Binds/スコープ設計"
emoji: "🧩"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt Module**（@Module、@Provides、@Binds、@InstallIn、スコープ設計）を解説します。

---

## 基本Module

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .build()

    @Provides
    fun provideTaskDao(database: AppDatabase): TaskDao = database.taskDao()

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
}
```

---

## @Binds

```kotlin
interface TaskRepository {
    fun getTasks(): Flow<List<Task>>
    suspend fun insert(task: Task)
}

class TaskRepositoryImpl @Inject constructor(
    private val taskDao: TaskDao
) : TaskRepository {
    override fun getTasks() = taskDao.getAll()
    override suspend fun insert(task: Task) = taskDao.insert(task)
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindTaskRepository(impl: TaskRepositoryImpl): TaskRepository
}
```

---

## スコープ設計

```kotlin
// Singleton — アプリ全体で1インスタンス
@Module
@InstallIn(SingletonComponent::class)
object SingletonModule {
    @Provides
    @Singleton
    fun provideAnalytics(): Analytics = AnalyticsImpl()
}

// ActivityScoped — Activity単位
@Module
@InstallIn(ActivityComponent::class)
object ActivityModule {
    @Provides
    @ActivityScoped
    fun provideNavigator(activity: Activity): Navigator = NavigatorImpl(activity)
}

// ViewModelScoped — ViewModel単位
@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {
    @Provides
    @ViewModelScoped
    fun provideUseCase(repo: TaskRepository): GetTasksUseCase = GetTasksUseCase(repo)
}
```

---

## まとめ

| アノテーション | 用途 |
|--------------|------|
| `@Module` | DI定義クラス |
| `@Provides` | インスタンス生成 |
| `@Binds` | インターフェースバインド |
| `@InstallIn` | スコープ指定 |

- `@Provides`で第三者ライブラリのインスタンス生成
- `@Binds`でインターフェース→実装のバインド（効率的）
- `@InstallIn`でライフサイクルスコープを決定
- `@Singleton`/`@ActivityScoped`/`@ViewModelScoped`で適切なスコープ選択

---

8種類のAndroidアプリテンプレート（Hilt DI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-viewmodel-2026)
- [Hilt Testing](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
- [Hilt Worker](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-worker-2026)
