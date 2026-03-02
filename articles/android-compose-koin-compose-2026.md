---
title: "Koin + Compose完全ガイド — koinViewModel/Module定義/スコープ"
emoji: "🧊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "koin"]
published: true
---

## この記事で学べること

**Koin + Compose**（koinViewModel、Module定義、スコープ、テスト）を解説します。

---

## Module定義

```kotlin
val appModule = module {
    single<AppDatabase> {
        Room.databaseBuilder(get(), AppDatabase::class.java, "app.db").build()
    }
    single<UserRepository> { UserRepositoryImpl(get<AppDatabase>().userDao()) }
    single<ApiService> { Retrofit.Builder().baseUrl(BASE_URL).build().create() }
    viewModel { UserListViewModel(get()) }
    viewModel { (userId: Long) -> UserDetailViewModel(userId, get()) }
}

// Application
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModule)
        }
    }
}
```

---

## Compose連携

```kotlin
@Composable
fun UserListScreen(viewModel: UserListViewModel = koinViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()

    LazyColumn {
        items(users) { user ->
            ListItem(
                headlineContent = { Text(user.name) },
                supportingContent = { Text(user.email) }
            )
        }
    }
}

// パラメータ付きViewModel
@Composable
fun UserDetailScreen(userId: Long) {
    val viewModel: UserDetailViewModel = koinViewModel { parametersOf(userId) }
    val user by viewModel.user.collectAsStateWithLifecycle()

    user?.let {
        Column(Modifier.padding(16.dp)) {
            Text(it.name, style = MaterialTheme.typography.headlineMedium)
            Text(it.email)
        }
    }
}
```

---

## テスト

```kotlin
class UserViewModelTest : KoinTest {
    private val viewModel: UserListViewModel by inject()

    @get:Rule
    val koinRule = KoinTestRule.create {
        modules(module {
            single<UserRepository> { FakeUserRepository() }
            viewModel { UserListViewModel(get()) }
        })
    }

    @Test
    fun loadUsers() = runTest {
        val users = viewModel.users.first()
        assertThat(users).isNotEmpty()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `koinViewModel` | ViewModel取得 |
| `module { }` | DI定義 |
| `single` | シングルトン |
| `parametersOf` | パラメータ注入 |

- `koinViewModel()`でCompose内からViewModel取得
- `module { }`でDSLベースのDI定義
- `parametersOf`でNavigation引数をViewModel注入
- アノテーション不要のシンプルなDI

---

8種類のAndroidアプリテンプレート（DI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt/DI](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-2026)
- [Hiltテスト](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
- [ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-savedstate-2026)
