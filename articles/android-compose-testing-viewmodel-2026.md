---
title: "ViewModel単体テストガイド — Turbine/MockK/TestDispatcher"
emoji: "🧪"
type: "tech"
topics: ["android", "kotlin", "testing", "viewmodel"]
published: true
---

## この記事で学べること

ViewModelの**単体テスト**（Turbine、MockK、TestDispatcher）を解説します。

---

## セットアップ

```kotlin
dependencies {
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.1")
    testImplementation("app.cash.turbine:turbine:1.2.0")
    testImplementation("io.mockk:mockk:1.13.14")
    testImplementation("junit:junit:4.13.2")
}
```

---

## 基本のテスト

```kotlin
class UserViewModelTest {

    private val testDispatcher = UnconfinedTestDispatcher()
    private val repository = mockk<UserRepository>()
    private lateinit var viewModel: UserViewModel

    @Before
    fun setup() {
        Dispatchers.setMain(testDispatcher)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun `loadUsers success`() = runTest {
        val users = listOf(User(1, "Alice"), User(2, "Bob"))
        coEvery { repository.getUsers() } returns users

        viewModel = UserViewModel(repository)

        viewModel.uiState.test {
            assertEquals(UiState.Loading, awaitItem())
            assertEquals(UiState.Success(users), awaitItem())
        }
    }

    @Test
    fun `loadUsers error`() = runTest {
        coEvery { repository.getUsers() } throws IOException("ネットワークエラー")

        viewModel = UserViewModel(repository)

        viewModel.uiState.test {
            assertEquals(UiState.Loading, awaitItem())
            val error = awaitItem()
            assertTrue(error is UiState.Error)
        }
    }
}
```

---

## Turbineでの Flow テスト

```kotlin
@Test
fun `search with debounce`() = runTest {
    coEvery { repository.search(any()) } returns listOf(Item(1, "Result"))

    viewModel = SearchViewModel(repository)

    viewModel.results.test {
        assertEquals(emptyList<Item>(), awaitItem()) // 初期値

        viewModel.updateQuery("test")
        advanceTimeBy(300) // debounce待ち

        assertEquals(listOf(Item(1, "Result")), awaitItem())
    }
}

@Test
fun `filter changes update list`() = runTest {
    val allItems = listOf(
        Item(1, "Apple", "fruit"),
        Item(2, "Carrot", "vegetable")
    )
    coEvery { repository.getAll() } returns flowOf(allItems)

    viewModel = ItemViewModel(repository)

    viewModel.filteredItems.test {
        assertEquals(allItems, awaitItem()) // 全件

        viewModel.selectCategory("fruit")
        assertEquals(listOf(allItems[0]), awaitItem()) // フィルター後
    }
}
```

---

## MockKの使い方

```kotlin
@Test
fun `createUser calls repository`() = runTest {
    coEvery { repository.createUser(any()) } returns User(1, "Alice")

    viewModel = UserViewModel(repository)
    viewModel.createUser("Alice", "alice@example.com")

    coVerify { repository.createUser(CreateUserRequest("Alice", "alice@example.com")) }
}

@Test
fun `deleteUser with confirmation`() = runTest {
    coEvery { repository.deleteUser(1) } just Runs

    viewModel = UserViewModel(repository)
    viewModel.deleteUser(1)

    coVerify(exactly = 1) { repository.deleteUser(1) }
}
```

---

## SavedStateHandleのテスト

```kotlin
@Test
fun `SavedStateHandle preserves query`() = runTest {
    val savedState = SavedStateHandle(mapOf("query" to "initial"))
    viewModel = SearchViewModel(savedState, repository)

    assertEquals("initial", viewModel.query.value)

    viewModel.updateQuery("updated")
    assertEquals("updated", viewModel.query.value)
    assertEquals("updated", savedState["query"])
}
```

---

## まとめ

- `UnconfinedTestDispatcher`で同期的にテスト実行
- `Turbine`の`test{}`でStateFlow/SharedFlowを検証
- `MockK`の`coEvery`/`coVerify`でsuspend関数をモック
- `advanceTimeBy()`でdebounce等の時間を進める
- `SavedStateHandle`は直接コンストラクタで初期値指定
- `Dispatchers.setMain`/`resetMain`は`@Before`/`@After`で設定

---

8種類のAndroidアプリテンプレート（テスト設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [UIテスト完全ガイド](https://zenn.dev/myougatheaxo/articles/android-testing-compose-2026)
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-screenshot-2026)
- [CI/CDガイド](https://zenn.dev/myougatheaxo/articles/android-cicd-github-actions-2026)
