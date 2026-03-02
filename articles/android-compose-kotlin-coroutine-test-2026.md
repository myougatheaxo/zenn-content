---
title: "Kotlin Coroutine Test完全ガイド — runTest/TestDispatcher/advanceTime/Turbine"
emoji: "⏱️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Kotlin Coroutine Test**（runTest、TestDispatcher、仮想時間制御、Turbine）を解説します。

---

## runTest基本

```kotlin
class UserRepositoryTest {
    @Test
    fun `fetch user returns data`() = runTest {
        val api = FakeUserApi()
        val repository = UserRepository(api)

        val user = repository.getUser("123")

        assertEquals("テストユーザー", user.name)
    }

    @Test
    fun `delay is skipped in runTest`() = runTest {
        val start = currentTime
        delay(10_000) // 仮想的にスキップ
        assertEquals(10_000, currentTime - start)
        // 実際の時間はほぼ0ms
    }
}
```

---

## TestDispatcher

```kotlin
class ViewModelTest {
    @OptIn(ExperimentalCoroutinesApi::class)
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun `loading state transitions`() = runTest {
        val viewModel = UserViewModel(FakeRepository())

        // 初期状態
        assertEquals(UiState.Loading, viewModel.uiState.value)

        // 仮想時間を進める
        advanceUntilIdle()

        // データ読み込み完了
        assertTrue(viewModel.uiState.value is UiState.Success)
    }
}

// MainDispatcherRule
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

## Turbineでflowテスト

```kotlin
@Test
fun `search flow emits results`() = runTest {
    val viewModel = SearchViewModel(FakeSearchRepository())

    viewModel.results.test {
        // 初期値
        assertEquals(emptyList<String>(), awaitItem())

        // 検索実行
        viewModel.search("kotlin")

        // debounce待ち
        advanceTimeBy(500)

        // 結果取得
        val results = awaitItem()
        assertTrue(results.isNotEmpty())

        cancelAndIgnoreRemainingEvents()
    }
}

@Test
fun `error handling in flow`() = runTest {
    val viewModel = UserViewModel(ErrorRepository())

    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem())
        assertTrue(awaitItem() is UiState.Error)
        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `runTest` | コルーチンテスト実行 |
| `TestDispatcher` | テスト用ディスパッチャー |
| `advanceUntilIdle` | 全コルーチン完了待ち |
| `Turbine` | Flowテスト |

- `runTest`でコルーチンを同期的にテスト（delayは仮想スキップ）
- `MainDispatcherRule`でMain dispatcherをテスト用に差し替え
- `advanceTimeBy`/`advanceUntilIdle`で仮想時間を制御
- Turbineの`test`でFlow値の変化を順序テスト

---

8種類のAndroidアプリテンプレート（テスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Scope](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-scope-2026)
- [Coroutine Dispatcher](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-dispatcher-2026)
- [Kotlin FlowTest](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-flow-test-2026)
