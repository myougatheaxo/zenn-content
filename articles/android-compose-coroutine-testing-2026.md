---
title: "コルーチンテスト完全ガイド — runTest/Turbine/TestDispatcher"
emoji: "🧪"
type: "tech"
topics: ["android", "kotlin", "coroutines", "testing"]
published: true
---

## この記事で学べること

**コルーチンテスト**（runTest、Turbine、TestDispatcher、Flow テスト、タイムアウト制御）を解説します。

---

## runTest 基本

```kotlin
class UserRepositoryTest {
    @Test
    fun `ユーザー取得が正常に動作する`() = runTest {
        val fakeApi = FakeUserApi()
        val repository = UserRepository(fakeApi)

        val user = repository.getUser("123")

        assertEquals("Test User", user.name)
    }
}

// runTest は自動的にテスト用ディスパッチャを使用
// delay() はスキップされる（仮想時間）
@Test
fun `delay付き処理のテスト`() = runTest {
    val startTime = currentTime
    delay(1000L)
    assertEquals(1000L, currentTime - startTime) // 仮想時間で即完了
}
```

---

## TestDispatcher

```kotlin
class ViewModelTest {
    // StandardTestDispatcher: 明示的にadvanceまで実行しない
    @Test
    fun `StandardTestDispatcher使用`() = runTest {
        val dispatcher = StandardTestDispatcher(testScheduler)
        val viewModel = MyViewModel(dispatcher)

        viewModel.loadData()
        advanceUntilIdle() // コルーチン完了まで進める

        assertEquals(UiState.Success, viewModel.state.value)
    }

    // UnconfinedTestDispatcher: 即座に実行
    @Test
    fun `UnconfinedTestDispatcher使用`() = runTest {
        val dispatcher = UnconfinedTestDispatcher(testScheduler)
        val viewModel = MyViewModel(dispatcher)

        viewModel.loadData() // 即座に完了
        assertEquals(UiState.Success, viewModel.state.value)
    }
}

// ViewModel側: Dispatcherを注入可能に
class MyViewModel(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : ViewModel() {
    fun loadData() {
        viewModelScope.launch(ioDispatcher) { /* ... */ }
    }
}
```

---

## Turbine（Flow テスト）

```kotlin
// build.gradle: testImplementation("app.cash.turbine:turbine:1.1.0")

@Test
fun `StateFlowの変化をテスト`() = runTest {
    val viewModel = SearchViewModel(FakeRepository())

    viewModel.results.test {
        assertEquals(emptyList(), awaitItem()) // 初期値

        viewModel.search("kotlin")
        assertEquals(UiState.Loading, awaitItem())
        assertEquals(UiState.Success(listOf("Kotlin")), awaitItem())

        cancelAndIgnoreRemainingEvents()
    }
}

@Test
fun `エラーFlowのテスト`() = runTest {
    val failingRepo = FailingRepository()

    failingRepo.getData().test {
        val error = awaitError()
        assertIs<IOException>(error)
    }
}

@Test
fun `複数Flowの同時テスト`() = runTest {
    val viewModel = DashboardViewModel()

    turbineScope {
        val userTurbine = viewModel.user.testIn(backgroundScope)
        val statsTurbine = viewModel.stats.testIn(backgroundScope)

        assertEquals(null, userTurbine.awaitItem())
        assertEquals(Stats.Empty, statsTurbine.awaitItem())

        viewModel.refresh()
        assertNotNull(userTurbine.awaitItem())
        assertNotEquals(Stats.Empty, statsTurbine.awaitItem())

        userTurbine.cancel()
        statsTurbine.cancel()
    }
}
```

---

## Dispatcher差し替えルール

```kotlin
class MainDispatcherRule(
    val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

class MyViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun `Dispatchers_Main使用コードのテスト`() = runTest {
        val viewModel = MyViewModel() // 内部でDispatchers.Main使用
        viewModel.loadData()
        // MainがTestDispatcherに差し替え済み
    }
}
```

---

## まとめ

| ツール | 用途 |
|--------|------|
| `runTest` | コルーチンテストの基盤 |
| `StandardTestDispatcher` | 明示的な実行制御 |
| `UnconfinedTestDispatcher` | 即座実行 |
| `Turbine` | Flowの値検証 |
| `MainDispatcherRule` | Main差し替え |

- `runTest`で仮想時間によるテスト高速化
- `Turbine`でFlowの値を順番に検証
- `MainDispatcherRule`でViewModel テストを安定化
- Dispatcher注入でテスタビリティ確保

---

8種類のAndroidアプリテンプレート（テスト設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テストRobotパターン](https://zenn.dev/myougatheaxo/articles/android-compose-testing-robot-pattern-2026)
- [Hiltテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-hilt-2026)
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-screenshot-comparison-2026)
