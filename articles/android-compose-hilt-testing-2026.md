---
title: "Hiltテスト完全ガイド — HiltAndroidTest/TestModule/Fake注入"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hiltテスト**（HiltAndroidTest、テスト用Module差し替え、Fakeリポジトリ注入）を解説します。

---

## HiltAndroidTest

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class UserViewModelTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)

    @Inject lateinit var repository: UserRepository

    @Before
    fun setup() {
        hiltRule.inject()
    }

    @Test
    fun loadUsers_returnsUsers() = runTest {
        val users = repository.getUsers().first()
        assertThat(users).isNotEmpty()
    }
}
```

---

## TestModule

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class]
)
object FakeRepositoryModule {
    @Provides
    @Singleton
    fun provideUserRepository(): UserRepository = FakeUserRepository()
}

class FakeUserRepository : UserRepository {
    private val users = MutableStateFlow(
        listOf(User(1, "テスト太郎"), User(2, "テスト花子"))
    )

    override fun getUsers(): Flow<List<User>> = users
    override suspend fun addUser(user: User) {
        users.update { it + user }
    }
    override suspend fun deleteUser(id: Long) {
        users.update { list -> list.filter { it.id != id } }
    }
}
```

---

## Compose UIテスト + Hilt

```kotlin
@HiltAndroidTest
class UserScreenTest {
    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeRule = createAndroidComposeRule<MainActivity>()

    @Before
    fun setup() {
        hiltRule.inject()
    }

    @Test
    fun userList_displaysUsers() {
        composeRule.onNodeWithText("テスト太郎").assertIsDisplayed()
        composeRule.onNodeWithText("テスト花子").assertIsDisplayed()
    }

    @Test
    fun addUser_showsNewUser() {
        composeRule.onNodeWithContentDescription("追加").performClick()
        composeRule.onNodeWithTag("name_input").performTextInput("新しいユーザー")
        composeRule.onNodeWithText("保存").performClick()
        composeRule.onNodeWithText("新しいユーザー").assertIsDisplayed()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@HiltAndroidTest` | Hiltテスト有効化 |
| `@TestInstallIn` | Module差し替え |
| `HiltAndroidRule` | DI初期化 |
| `FakeRepository` | テスト用実装 |

- `@HiltAndroidTest`でDIありのテスト環境
- `@TestInstallIn`で本番Moduleをテスト用に差し替え
- Fakeリポジトリで外部依存を排除
- Compose UIテストとHiltの組み合わせ

---

8種類のAndroidアプリテンプレート（テスト実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt/DI](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-2026)
- [UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ui-test-2026)
- [Robolectric](https://zenn.dev/myougatheaxo/articles/android-compose-robolectric-2026)
