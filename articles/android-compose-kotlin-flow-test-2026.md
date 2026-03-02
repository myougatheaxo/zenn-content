---
title: "Kotlin Flow Test完全ガイド — Turbine/TestScope/StateFlow検証/SharedFlowテスト"
emoji: "🌊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "flow"]
published: true
---

## この記事で学べること

**Kotlin Flow Test**（Turbine、StateFlow検証、SharedFlowテスト、エラーハンドリングテスト）を解説します。

---

## StateFlowテスト

```kotlin
class CounterViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun `increment updates count`() = runTest {
        val viewModel = CounterViewModel()

        viewModel.count.test {
            assertEquals(0, awaitItem()) // 初期値

            viewModel.increment()
            assertEquals(1, awaitItem())

            viewModel.increment()
            assertEquals(2, awaitItem())

            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## 複雑なFlowテスト

```kotlin
@Test
fun `search with debounce`() = runTest {
    val fakeApi = FakeSearchApi()
    val viewModel = SearchViewModel(fakeApi)

    viewModel.searchResults.test {
        assertEquals(emptyList<String>(), awaitItem())

        // 高速入力（debounce 300ms）
        viewModel.onQueryChange("k")
        viewModel.onQueryChange("ko")
        viewModel.onQueryChange("kot")
        advanceTimeBy(100) // まだdebounce中
        expectNoEvents() // イベントなし

        advanceTimeBy(300) // debounce完了
        val results = awaitItem()
        assertEquals(listOf("Kotlin", "Kotlin Coroutines"), results)

        cancelAndIgnoreRemainingEvents()
    }
}

@Test
fun `flow error recovery`() = runTest {
    val repo = FlakyRepository(failFirst = true)
    val viewModel = DataViewModel(repo)

    viewModel.data.test {
        // Loading
        assertEquals(UiState.Loading, awaitItem())
        // エラー
        assertTrue(awaitItem() is UiState.Error)

        // リトライ
        viewModel.retry()
        assertEquals(UiState.Loading, awaitItem())
        assertTrue(awaitItem() is UiState.Success)

        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## SharedFlowテスト

```kotlin
@Test
fun `one-time events via SharedFlow`() = runTest {
    val viewModel = FormViewModel()

    viewModel.events.test {
        viewModel.submit("valid@example.com", "password123")
        val event = awaitItem()
        assertTrue(event is FormEvent.NavigateToHome)

        viewModel.submit("", "")
        val errorEvent = awaitItem()
        assertTrue(errorEvent is FormEvent.ShowError)

        cancelAndIgnoreRemainingEvents()
    }
}

@Test
fun `combine multiple flows`() = runTest {
    val viewModel = DashboardViewModel(FakeRepo())

    viewModel.dashboardState.test {
        val state = awaitItem()
        assertTrue(state.users.isNotEmpty())
        assertTrue(state.stats.totalCount > 0)

        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Turbine test` | Flow値の順序検証 |
| `awaitItem` | 次の値を取得 |
| `expectNoEvents` | イベント無しを確認 |
| `advanceTimeBy` | 仮想時間操作 |

- Turbineで`awaitItem()`を使いFlow値を順序的にテスト
- `expectNoEvents()`でdebounce等の未発行を確認
- `advanceTimeBy`で仮想時間を進めてタイミングテスト
- `cancelAndIgnoreRemainingEvents()`でテスト終了

---

8種類のAndroidアプリテンプレート（テスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin CoroutineTest](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-coroutine-test-2026)
- [Flow Buffer](https://zenn.dev/myougatheaxo/articles/android-compose-flow-buffer-2026)
- [Flow Transform](https://zenn.dev/myougatheaxo/articles/android-compose-flow-transform-2026)
