---
title: "ViewModelテスト完全ガイド — StateFlow検証/UseCase mock/Turbine"
emoji: "🔬"
type: "tech"
topics: ["android", "kotlin", "testing", "viewmodel"]
published: true
---

## この記事で学べること

**ViewModelテスト**（StateFlow検証、UseCase mock、MainDispatcherRule、Turbine統合）を解説します。

---

## 基本テスト

```kotlin
class HomeViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var viewModel: HomeViewModel
    private val fakeRepository = FakeArticleRepository()

    @Before
    fun setup() {
        viewModel = HomeViewModel(
            getArticlesUseCase = GetArticlesUseCase(fakeRepository)
        )
    }

    @Test
    fun `初期状態はLoading`() {
        assertEquals(UiState.Loading, viewModel.uiState.value)
    }

    @Test
    fun `データ取得成功でSuccessになる`() = runTest {
        fakeRepository.setArticles(listOf(Article("1", "Test")))

        viewModel.loadArticles()
        advanceUntilIdle()

        val state = viewModel.uiState.value
        assertIs<UiState.Success>(state)
        assertEquals(1, state.data.size)
    }

    @Test
    fun `エラー時はErrorになる`() = runTest {
        fakeRepository.setShouldFail(true)

        viewModel.loadArticles()
        advanceUntilIdle()

        assertIs<UiState.Error>(viewModel.uiState.value)
    }
}
```

---

## Turbineで状態遷移テスト

```kotlin
@Test
fun `検索の状態遷移を検証`() = runTest {
    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem()) // 初期状態

        viewModel.search("kotlin")
        assertEquals(UiState.Loading, awaitItem()) // 検索中

        val success = awaitItem()
        assertIs<UiState.Success>(success)
        assertTrue(success.data.isNotEmpty())

        cancelAndIgnoreRemainingEvents()
    }
}

@Test
fun `イベント発火を検証`() = runTest {
    viewModel.events.test {
        viewModel.submitForm("test@example.com")

        val event = awaitItem()
        assertIs<UiEvent.NavigateToHome>(event)

        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## Fake Repository

```kotlin
class FakeArticleRepository : ArticleRepository {
    private var articles = listOf<Article>()
    private var shouldFail = false

    fun setArticles(list: List<Article>) { articles = list }
    fun setShouldFail(fail: Boolean) { shouldFail = fail }

    override fun getArticles(): Flow<List<Article>> = flow {
        if (shouldFail) throw IOException("Network error")
        emit(articles)
    }

    override suspend fun getArticle(id: String): Article {
        if (shouldFail) throw IOException("Network error")
        return articles.first { it.id == id }
    }
}
```

---

## SavedStateHandle テスト

```kotlin
@Test
fun `SavedStateHandleから初期値を復元`() {
    val savedState = SavedStateHandle(mapOf("query" to "kotlin"))
    val viewModel = SearchViewModel(savedState, fakeRepository)

    assertEquals("kotlin", viewModel.query.value)
}

@Test
fun `入力値がSavedStateHandleに保存される`() {
    val savedState = SavedStateHandle()
    val viewModel = SearchViewModel(savedState, fakeRepository)

    viewModel.updateQuery("compose")

    assertEquals("compose", savedState.get<String>("query"))
}
```

---

## まとめ

| テスト対象 | ツール |
|-----------|--------|
| StateFlow | `Turbine` / `.value` |
| コルーチン | `runTest` + `advanceUntilIdle` |
| Dispatcher | `MainDispatcherRule` |
| SavedState | `SavedStateHandle(mapOf())` |
| 依存関係 | Fake Repository |

- `MainDispatcherRule`でDispatchers.Mainを差し替え
- `Turbine`でStateFlowの状態遷移を順序テスト
- Fake Repositoryで外部依存を排除
- SavedStateHandleは直接コンストラクタに渡してテスト

---

8種類のAndroidアプリテンプレート（テスト設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [コルーチンテスト](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-testing-2026)
- [Composeテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [Hiltテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-hilt-2026)
