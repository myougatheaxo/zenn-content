---
title: "ViewModel単体テスト完全ガイド — Turbine/MockK/TestDispatcher"
emoji: "🧪"
type: "tech"
topics: ["android", "kotlin", "testing", "viewmodel"]
published: true
---

## この記事で学べること

**ViewModelの単体テスト**（Turbine、MockK、TestDispatcher、StateFlowテスト）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    testImplementation("io.mockk:mockk:1.13.12")
    testImplementation("app.cash.turbine:turbine:1.1.0")
    testImplementation("androidx.arch.core:core-testing:2.2.0")
}
```

---

## MainDispatcherRule

```kotlin
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

---

## 基本のViewModelテスト

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val users = repository.getUsers()
                _uiState.value = UiState.Success(users)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}

// テスト
class UserViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private val repository = mockk<UserRepository>()
    private lateinit var viewModel: UserViewModel

    @Before
    fun setup() {
        viewModel = UserViewModel(repository)
    }

    @Test
    fun `loadUsers success updates state`() = runTest {
        val users = listOf(User("1", "Alice"), User("2", "Bob"))
        coEvery { repository.getUsers() } returns users

        viewModel.uiState.test {
            assertEquals(UiState.Idle, awaitItem())

            viewModel.loadUsers()

            assertEquals(UiState.Loading, awaitItem())
            assertEquals(UiState.Success(users), awaitItem())
        }
    }

    @Test
    fun `loadUsers error updates state`() = runTest {
        coEvery { repository.getUsers() } throws IOException("Network error")

        viewModel.uiState.test {
            assertEquals(UiState.Idle, awaitItem())

            viewModel.loadUsers()

            assertEquals(UiState.Loading, awaitItem())
            val error = awaitItem()
            assertTrue(error is UiState.Error)
            assertEquals("Network error", (error as UiState.Error).message)
        }
    }
}
```

---

## MockKパターン

```kotlin
// coEvery: suspend関数のモック
coEvery { repository.getUser(any()) } returns User("1", "Test")

// every: 通常関数のモック
every { repository.observeUsers() } returns flowOf(listOf(User("1", "Test")))

// 引数キャプチャ
val slot = slot<String>()
coEvery { repository.getUser(capture(slot)) } returns User("1", "Test")
viewModel.loadUser("123")
assertEquals("123", slot.captured)

// 呼び出し回数検証
coVerify(exactly = 1) { repository.getUsers() }
coVerify { repository.save(any()) wasNot Called }

// 順序検証
coVerifyOrder {
    repository.getUsers()
    repository.save(any())
}
```

---

## Turbine でFlowテスト

```kotlin
@Test
fun `search filters results`() = runTest {
    val allUsers = listOf(User("1", "Alice"), User("2", "Bob"), User("3", "Anna"))
    every { repository.observeUsers() } returns flowOf(allUsers)

    viewModel.searchResults.test {
        // 初期状態
        assertEquals(allUsers, awaitItem())

        // 検索
        viewModel.updateQuery("A")
        assertEquals(listOf(User("1", "Alice"), User("3", "Anna")), awaitItem())

        // クリア
        viewModel.updateQuery("")
        assertEquals(allUsers, awaitItem())

        cancelAndConsumeRemainingEvents()
    }
}

// SharedFlow（イベント）のテスト
@Test
fun `delete emits success event`() = runTest {
    coEvery { repository.delete(any()) } returns Unit

    viewModel.events.test {
        viewModel.deleteUser("1")

        val event = awaitItem()
        assertTrue(event is UiEvent.ShowSnackbar)
        assertEquals("削除しました", (event as UiEvent.ShowSnackbar).message)
    }
}
```

---

## SavedStateHandle テスト

```kotlin
@Test
fun `loads user from savedStateHandle`() = runTest {
    val savedState = SavedStateHandle(mapOf("userId" to "123"))
    coEvery { repository.getUser("123") } returns User("123", "Test")

    val viewModel = UserDetailViewModel(repository, savedState)

    viewModel.user.test {
        assertNull(awaitItem()) // 初期値
        assertEquals(User("123", "Test"), awaitItem())
    }
}
```

---

## まとめ

| ツール | 用途 |
|--------|------|
| `MainDispatcherRule` | テスト用Dispatchers.Main設定 |
| `MockK` | モック作成（coEvery/coVerify） |
| `Turbine` | Flow/StateFlowの値アサーション |
| `runTest` | コルーチンテスト |
| `UnconfinedTestDispatcher` | 即時実行ディスパッチャー |

- `coEvery`でsuspend関数、`every`で通常関数をモック
- `viewModel.uiState.test { }`でStateFlowの遷移を検証
- `SavedStateHandle`はコンストラクタで直接渡す
- `cancelAndConsumeRemainingEvents()`でリソース解放

---

8種類のAndroidアプリテンプレート（テスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ComposeTestRule](https://zenn.dev/myougatheaxo/articles/android-compose-testing-compose-rule-2026)
- [ViewModel + Hilt](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-screenshot-2026)
