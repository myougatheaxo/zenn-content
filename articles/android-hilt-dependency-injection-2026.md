---
title: "Hilt完全ガイド — AndroidのDI（依存性注入）をマスターする"
emoji: "💉"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt**を使った依存性注入（DI）の基本から、ViewModel・Room・Retrofit連携まで解説します。

---

## セットアップ

```kotlin
// project build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android") version "2.51" apply false
}

// app build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.51")
    ksp("com.google.dagger:hilt-compiler:2.51")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
}
```

---

## Application クラス

```kotlin
@HiltAndroidApp
class MyApp : Application()
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:name=".MyApp"
    ...>
```

---

## Activity

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyApp()
        }
    }
}
```

---

## Module（依存の提供）

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }

    @Provides
    fun provideTaskDao(db: AppDatabase): TaskDao = db.taskDao()

    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

---

## Repository（インターフェース + 実装）

```kotlin
interface TaskRepository {
    fun observeAll(): Flow<List<Task>>
    suspend fun add(task: Task)
    suspend fun delete(task: Task)
}

class TaskRepositoryImpl @Inject constructor(
    private val dao: TaskDao,
    private val api: ApiService
) : TaskRepository {
    override fun observeAll(): Flow<List<Task>> = dao.observeAll()

    override suspend fun add(task: Task) {
        dao.insert(task)
        api.syncTask(task)
    }

    override suspend fun delete(task: Task) {
        dao.delete(task)
    }
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

## ViewModel

```kotlin
@HiltViewModel
class TaskViewModel @Inject constructor(
    private val repository: TaskRepository
) : ViewModel() {

    val tasks: StateFlow<List<Task>> = repository.observeAll()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun addTask(title: String) {
        viewModelScope.launch {
            repository.add(Task(title = title))
        }
    }
}
```

---

## ComposeでViewModelを取得

```kotlin
@Composable
fun TaskScreen(
    viewModel: TaskViewModel = hiltViewModel()
) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()

    LazyColumn {
        items(tasks, key = { it.id }) { task ->
            TaskItem(task)
        }
    }
}
```

`hiltViewModel()` で自動注入。Navigation Composeとも連携。

---

## スコープ一覧

| スコープ | ライフサイクル |
|---------|-------------|
| `SingletonComponent` | アプリ全体 |
| `ActivityComponent` | Activity |
| `ViewModelComponent` | ViewModel |
| `FragmentComponent` | Fragment |
| `ServiceComponent` | Service |

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {
    @Provides
    @ViewModelScoped
    fun provideUseCase(repo: TaskRepository): GetTasksUseCase {
        return GetTasksUseCase(repo)
    }
}
```

---

## Qualifier（同じ型の区別）

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}

class TaskRepository @Inject constructor(
    private val dao: TaskDao,
    @IoDispatcher private val dispatcher: CoroutineDispatcher
) {
    // ...
}
```

---

## まとめ

- `@HiltAndroidApp` → Application、`@AndroidEntryPoint` → Activity
- `@Module` + `@Provides` で依存を提供
- `@Binds` でインターフェースと実装をバインド
- `@HiltViewModel` + `hiltViewModel()` でCompose連携
- `@Singleton` / `@ViewModelScoped` でスコープ管理
- `@Qualifier` で同じ型の依存を区別

---

8種類のAndroidアプリテンプレート（Hilt DI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Database完全ガイド](https://zenn.dev/myougatheaxo/articles/android-room-database-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
- [Clean Architecture実践ガイド](https://zenn.dev/myougatheaxo/articles/android-app-architecture-clean-2026)
