---
title: "Hilt依存性注入 完全ガイド — Module/Scope/テスト"
emoji: "💉"
type: "tech"
topics: ["android", "kotlin", "hilt", "di"]
published: true
---

## この記事で学べること

**Hilt**の依存性注入（Module定義、Scope管理、ViewModel注入、テスト差し替え）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts (app)
plugins {
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.51.1")
    ksp("com.google.dagger:hilt-compiler:2.51.1")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
}

// Application
@HiltAndroidApp
class MyApp : Application()

// Activity
@AndroidEntryPoint
class MainActivity : ComponentActivity()
```

---

## Module定義

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi {
        return retrofit.create(UserApi::class.java)
    }
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .fallbackToDestructiveMigration()
            .build()
    }

    @Provides
    fun provideUserDao(db: AppDatabase): UserDao = db.userDao()
}
```

---

## @Binds でインターフェース注入

```kotlin
interface UserRepository {
    fun observeUsers(): Flow<List<User>>
    suspend fun getUser(id: String): User
}

class UserRepositoryImpl @Inject constructor(
    private val dao: UserDao,
    private val api: UserApi
) : UserRepository {
    override fun observeUsers() = dao.observeAll()
    override suspend fun getUser(id: String) = api.getUser(id)
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

---

## ViewModel注入

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val userId: String = savedStateHandle["userId"] ?: ""

    val user = repository.observeUser(userId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}

// Compose
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val user by viewModel.user.collectAsStateWithLifecycle()
    // ...
}
```

---

## Scope一覧

```kotlin
// @Singleton → アプリ全体で1インスタンス
// @ActivityScoped → Activity単位
// @ViewModelScoped → ViewModel単位
// @ActivityRetainedScoped → 画面回転でも維持

@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {
    @Provides
    @ViewModelScoped
    fun provideUseCase(repository: UserRepository): GetUserUseCase {
        return GetUserUseCase(repository)
    }
}
```

---

## テストでの差し替え

```kotlin
@HiltAndroidTest
@UninstallModules(RepositoryModule::class)
class UserScreenTest {

    @get:Rule
    val hiltRule = HiltAndroidRule(this)

    @Module
    @InstallIn(SingletonComponent::class)
    abstract class TestModule {
        @Binds
        @Singleton
        abstract fun bindRepo(impl: FakeUserRepository): UserRepository
    }

    @Inject
    lateinit var repository: UserRepository

    @Before
    fun setup() {
        hiltRule.inject()
    }

    @Test
    fun showsUserList() {
        // FakeUserRepositoryが注入される
    }
}

class FakeUserRepository @Inject constructor() : UserRepository {
    override fun observeUsers() = flowOf(listOf(User("1", "Test")))
    override suspend fun getUser(id: String) = User(id, "Test")
}
```

---

## まとめ

| アノテーション | 用途 |
|--------------|------|
| `@HiltAndroidApp` | Application登録 |
| `@AndroidEntryPoint` | Activity/Fragment注入有効化 |
| `@HiltViewModel` | ViewModel注入 |
| `@Module` + `@InstallIn` | 依存関係の定義 |
| `@Provides` | インスタンス生成 |
| `@Binds` | インターフェース→実装のバインド |
| `@Singleton` | シングルトンスコープ |

- `hiltViewModel()`でCompose内からViewModel取得
- `@UninstallModules`でテスト時にモジュール差し替え
- インターフェースは`@Binds`、具体クラスは`@Provides`

---

8種類のAndroidアプリテンプレート（Hilt DI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Koin DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-koin-2026)
- [Clean Architecture](https://zenn.dev/myougatheaxo/articles/android-compose-clean-architecture-2026)
- [ViewModel + Hilt](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
