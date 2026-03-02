---
title: "Hiltテスト完全ガイド — @HiltAndroidTest/FakeModule/UIテスト連携"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Hiltテスト**（@HiltAndroidTest、TestModule差し替え、Compose UIテスト連携、カスタムTestRunner）を解説します。

---

## セットアップ

```kotlin
dependencies {
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.53.1")
    kspAndroidTest("com.google.dagger:hilt-android-compiler:2.53.1")
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
}
```

---

## カスタムTestRunner

```kotlin
// app/src/androidTest/
class HiltTestRunner : AndroidJUnitRunner() {
    override fun newApplication(cl: ClassLoader?, name: String?, context: Context?): Application {
        return super.newApplication(cl, HiltTestApplication::class.java.name, context)
    }
}

// build.gradle.kts
android {
    defaultConfig {
        testInstrumentationRunner = "com.example.HiltTestRunner"
    }
}
```

---

## Fake Moduleで差し替え

```kotlin
// 本番Module
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    @Provides
    @Singleton
    fun provideUserRepository(api: UserApi, dao: UserDao): UserRepository {
        return UserRepositoryImpl(api, dao)
    }
}

// テスト用Module（本番を差し替え）
@Module
@InstallIn(SingletonComponent::class)
object FakeRepositoryModule {
    @Provides
    @Singleton
    fun provideUserRepository(): UserRepository {
        return FakeUserRepository()
    }
}

class FakeUserRepository : UserRepository {
    var users = mutableListOf(User("1", "Test User", "test@example.com"))

    override fun getUsers(): Flow<List<User>> = flowOf(users)
    override suspend fun getUser(id: String) = users.first { it.id == id }
}
```

---

## UIテスト

```kotlin
@HiltAndroidTest
@UninstallModules(RepositoryModule::class)  // 本番Moduleを除外
class UserListScreenTest {
    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeRule = createAndroidComposeRule<MainActivity>()

    @Module
    @InstallIn(SingletonComponent::class)
    object TestModule {
        @Provides
        @Singleton
        fun provideUserRepository(): UserRepository = FakeUserRepository()
    }

    @Before
    fun setup() {
        hiltRule.inject()
    }

    @Test
    fun userListDisplaysUsers() {
        composeRule.onNodeWithText("Test User").assertIsDisplayed()
    }

    @Test
    fun clickUserNavigatesToDetail() {
        composeRule.onNodeWithText("Test User").performClick()
        composeRule.onNodeWithText("test@example.com").assertIsDisplayed()
    }
}
```

---

## ViewModelテスト

```kotlin
@HiltAndroidTest
class UserViewModelTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)

    @Inject lateinit var repository: UserRepository

    @Module
    @InstallIn(SingletonComponent::class)
    object TestModule {
        @Provides fun provideRepo(): UserRepository = FakeUserRepository()
    }

    @Before fun setup() { hiltRule.inject() }

    @Test
    fun loadUsersReturnsData() = runTest {
        val viewModel = UserViewModel(repository)
        viewModel.users.test {
            val users = awaitItem()
            assertEquals(1, users.size)
            assertEquals("Test User", users[0].name)
        }
    }
}
```

---

## まとめ

| 要素 | 役割 |
|------|------|
| `@HiltAndroidTest` | Hiltテスト有効化 |
| `@UninstallModules` | 本番Module除外 |
| `HiltAndroidRule` | テスト用DI初期化 |
| `FakeRepository` | テスト用実装 |

- `@UninstallModules`で本番のModuleをFakeに差し替え
- `HiltAndroidRule`でテスト時のDI初期化
- Compose UIテストとHiltを組み合わせ
- FakeRepositoryで外部依存を排除

---

8種類のAndroidアプリテンプレート（テスト設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-ui-2026)
- [ViewModelテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-viewmodel-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
