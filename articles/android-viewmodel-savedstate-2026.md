---
title: "ViewModel SavedStateHandleガイド — プロセスキル復帰対応"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "viewmodel"]
published: true
---

## この記事で学べること

ViewModelの**SavedStateHandle**を使ったプロセスキル復帰対応を解説します。

---

## なぜSavedStateHandleが必要か

```
通常のViewModel変数:
  メモリ上のみ → プロセスキルで消失

SavedStateHandle:
  Bundle経由で保存 → プロセスキル後も復元
```

---

## 基本の使い方

```kotlin
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // StateFlowとして取得（自動復元）
    val query: StateFlow<String> =
        savedStateHandle.getStateFlow("query", "")

    val selectedTab: StateFlow<Int> =
        savedStateHandle.getStateFlow("selectedTab", 0)

    fun updateQuery(newQuery: String) {
        savedStateHandle["query"] = newQuery
    }

    fun selectTab(index: Int) {
        savedStateHandle["selectedTab"] = index
    }
}
```

---

## Compose画面での使用

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel = viewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val selectedTab by viewModel.selectedTab.collectAsStateWithLifecycle()

    Column {
        OutlinedTextField(
            value = query,
            onValueChange = { viewModel.updateQuery(it) },
            label = { Text("検索") },
            modifier = Modifier.fillMaxWidth()
        )

        TabRow(selectedTabIndex = selectedTab) {
            listOf("すべて", "画像", "動画").forEachIndexed { index, title ->
                Tab(
                    selected = selectedTab == index,
                    onClick = { viewModel.selectTab(index) },
                    text = { Text(title) }
                )
            }
        }
    }
}
```

---

## Navigation引数との連携

```kotlin
class UserDetailViewModel(
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Navigationの引数を直接取得
    private val userId: String = checkNotNull(savedStateHandle["userId"])

    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init {
        loadUser(userId)
    }

    private fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val user = repository.getUser(id)
                _uiState.value = UiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "エラー")
            }
        }
    }
}
```

---

## リスト・オブジェクトの保存

```kotlin
class FilterViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // リストの保存（ArrayList<String>はBundle対応）
    val selectedTags: StateFlow<List<String>> =
        savedStateHandle.getStateFlow("tags", emptyList())

    fun toggleTag(tag: String) {
        val current = savedStateHandle.get<List<String>>("tags") ?: emptyList()
        savedStateHandle["tags"] = if (tag in current) {
            current - tag
        } else {
            current + tag
        }
    }

    // Parcelableオブジェクトの保存
    val sortOrder: StateFlow<String> =
        savedStateHandle.getStateFlow("sortOrder", "date_desc")

    fun setSortOrder(order: String) {
        savedStateHandle["sortOrder"] = order
    }
}
```

---

## Hilt連携

```kotlin
@HiltViewModel
class TodoViewModel @Inject constructor(
    private val repository: TodoRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    val filter: StateFlow<String> =
        savedStateHandle.getStateFlow("filter", "all")

    val todos: StateFlow<List<Todo>> = filter
        .flatMapLatest { filterValue ->
            when (filterValue) {
                "active" -> repository.getActiveTodos()
                "completed" -> repository.getCompletedTodos()
                else -> repository.getAllTodos()
            }
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun setFilter(newFilter: String) {
        savedStateHandle["filter"] = newFilter
    }
}
```

---

## テスト

```kotlin
@Test
fun `SavedStateHandle preserves state`() {
    val savedState = SavedStateHandle(mapOf("query" to "test"))
    val viewModel = SearchViewModel(savedState)

    assertEquals("test", viewModel.query.value)

    viewModel.updateQuery("new query")
    assertEquals("new query", viewModel.query.value)
}
```

---

## まとめ

- `SavedStateHandle`でプロセスキル後の状態復元
- `getStateFlow()`でリアクティブな状態管理
- `savedStateHandle["key"] = value`で保存
- Navigation引数も`SavedStateHandle`経由で取得
- Hilt連携で自動注入
- ユーザー入力（検索、フィルター、タブ）の保存に最適

---

8種類のAndroidアプリテンプレート（SavedStateHandle対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel/StateFlowガイド](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [状態管理ガイド](https://zenn.dev/myougatheaxo/articles/compose-state-management-2026)
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
