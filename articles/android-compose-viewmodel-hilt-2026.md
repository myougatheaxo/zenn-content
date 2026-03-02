---
title: "ViewModel + Hilt完全ガイド — Compose統合パターン"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**ViewModel + Hilt**のCompose統合パターン（hiltViewModel、AssistedInject、テスト）を解説します。

---

## 基本セットアップ

```kotlin
dependencies {
    implementation("com.google.dagger:hilt-android:2.51.1")
    ksp("com.google.dagger:hilt-android-compiler:2.51.1")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
}
```

---

## @HiltViewModel基本

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users = _users.asStateFlow()

    private val _isLoading = MutableStateFlow(false)
    val isLoading = _isLoading.asStateFlow()

    init {
        loadUsers()
    }

    fun loadUsers() {
        viewModelScope.launch {
            _isLoading.value = true
            _users.value = repository.getAll()
            _isLoading.value = false
        }
    }
}

// Composeで使用
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()

    // UI
}
```

---

## Navigation引数の受け取り

```kotlin
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    private val repository: UserRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Navigation引数をSavedStateHandleから取得
    private val userId: String = checkNotNull(savedStateHandle["userId"])

    val user = repository.getUserFlow(userId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}

// NavHost
composable("user/{userId}") { backStackEntry ->
    UserDetailScreen() // hiltViewModelが自動でSavedStateHandle注入
}
```

---

## Hilt Module定義

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .build()
    }

    @Provides
    fun provideUserDao(db: AppDatabase): UserDao = db.userDao()
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
}
```

---

## 共有ViewModel

```kotlin
// 親画面と子画面でViewModelを共有
@Composable
fun ParentScreen(navController: NavHostController) {
    // NavBackStackEntryのスコープでViewModelを共有
    val parentEntry = remember(navController) {
        navController.getBackStackEntry("parent")
    }
    val sharedViewModel: SharedViewModel = hiltViewModel(parentEntry)

    NavHost(navController, startDestination = "child1") {
        composable("child1") {
            val vm: SharedViewModel = hiltViewModel(parentEntry)
            ChildScreen1(vm)
        }
        composable("child2") {
            val vm: SharedViewModel = hiltViewModel(parentEntry)
            ChildScreen2(vm)
        }
    }
}
```

---

## まとめ

- `@HiltViewModel`+`@Inject constructor`でDI対応ViewModel
- `hiltViewModel()`でCompose内からViewModel取得
- `SavedStateHandle`でNavigation引数を自動受け取り
- `@Binds`でインターフェース↔実装のバインド
- `@Provides`でサードパーティライブラリのインスタンス提供
- `hiltViewModel(parentEntry)`で画面間ViewModel共有

---

8種類のAndroidアプリテンプレート（Hilt DI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
- [Koin DIガイド](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-koin-2026)
- [Clean Architecture](https://zenn.dev/myougatheaxo/articles/android-compose-clean-architecture-2026)
