---
title: "Koin DI + Compose連携ガイド — Hiltの代替"
emoji: "💉"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "koin"]
published: true
---

## この記事で学べること

**Koin**を使ったCompose向けDI（依存性注入）をHiltとの比較で解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("io.insert-koin:koin-android:3.5.6")
    implementation("io.insert-koin:koin-androidx-compose:3.5.6")
}
```

---

## モジュール定義

```kotlin
val appModule = module {
    // Singleton
    single<UserRepository> { UserRepositoryImpl(get()) }

    // Factory（毎回新しいインスタンス）
    factory { UserValidator() }

    // ViewModel
    viewModel { UserListViewModel(get()) }
    viewModel { (userId: String) -> UserDetailViewModel(userId, get()) }
}

val networkModule = module {
    single {
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor())
            .build()
    }

    single {
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(get())
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    single { get<Retrofit>().create(UserApi::class.java) }
}
```

---

## Application設定

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModule, networkModule)
        }
    }
}
```

---

## Compose画面での利用

```kotlin
@Composable
fun UserListScreen(
    viewModel: UserListViewModel = koinViewModel()
) {
    val users by viewModel.users.collectAsStateWithLifecycle()

    LazyColumn {
        items(users) { user ->
            UserItem(user)
        }
    }
}

// パラメータ付きViewModel
@Composable
fun UserDetailScreen(userId: String) {
    val viewModel: UserDetailViewModel = koinViewModel { parametersOf(userId) }
    val user by viewModel.user.collectAsStateWithLifecycle()

    user?.let { UserProfile(it) }
}
```

---

## Hiltとの比較

```kotlin
// Hilt: アノテーションベース（コンパイル時チェック）
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()

// Koin: DSLベース（ランタイム解決）
val module = module {
    viewModel { UserViewModel(get()) }
}

// Hilt: モジュール
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}

// Koin: モジュール
val module = module {
    single<UserRepository> { UserRepositoryImpl(get()) }
}
```

---

## まとめ

| 特徴 | Hilt | Koin |
|------|------|------|
| 解決タイミング | コンパイル時 | ランタイム |
| セットアップ | アノテーション多め | DSLでシンプル |
| エラー検出 | コンパイル時 | 実行時 |
| KMP対応 | ❌ | ✅ |
| 学習コスト | 中 | 低 |

- `koinViewModel()`でCompose内のViewModel注入
- `get()`で依存関係を自動解決
- `single`/`factory`でスコープ制御
- KMPプロジェクトではKoinが有利

---

8種類のAndroidアプリテンプレート（DI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
- [Repositoryパターン](https://zenn.dev/myougatheaxo/articles/kotlin-interface-patterns-2026)
- [ViewModelテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-viewmodel-2026)
