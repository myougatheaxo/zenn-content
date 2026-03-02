---
title: "Compose ViewModel SavedState完全ガイド — SavedStateHandle/プロセスキル対応"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "viewmodel"]
published: true
---

## この記事で学べること

**Compose ViewModel SavedState**（SavedStateHandle、プロセスキル対応、Navigation引数、状態復元）を解説します。

---

## SavedStateHandle基本

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // プロセスキルしても復元される
    val searchQuery = savedStateHandle.getStateFlow("query", "")
    val selectedFilter = savedStateHandle.getStateFlow("filter", "all")

    fun updateQuery(query: String) {
        savedStateHandle["query"] = query
    }

    fun updateFilter(filter: String) {
        savedStateHandle["filter"] = filter
    }
}

@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.searchQuery.collectAsStateWithLifecycle()
    val filter by viewModel.selectedFilter.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = query,
            onValueChange = { viewModel.updateQuery(it) },
            label = { Text("検索") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        FilterChipRow(selected = filter, onSelect = { viewModel.updateFilter(it) })
    }
}
```

---

## Navigation引数の受け取り

```kotlin
@HiltViewModel
class DetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    // Navigationの引数を自動取得
    val itemId: String = savedStateHandle["itemId"] ?: ""
    val category: String = savedStateHandle["category"] ?: ""

    private val _item = MutableStateFlow<Item?>(null)
    val item: StateFlow<Item?> = _item

    init {
        viewModelScope.launch {
            _item.value = repository.getItem(itemId)
        }
    }
}

// NavGraph設定
composable(
    route = "detail/{itemId}?category={category}",
    arguments = listOf(
        navArgument("itemId") { type = NavType.StringType },
        navArgument("category") { defaultValue = "general" }
    )
) {
    DetailScreen() // hiltViewModel()が自動でSavedStateHandleに引数を注入
}
```

---

## リスト選択状態の保存

```kotlin
@HiltViewModel
class ListViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    val selectedIds: StateFlow<Set<String>> = savedStateHandle
        .getStateFlow("selectedIds", emptySet<String>())

    val scrollPosition: StateFlow<Int> = savedStateHandle
        .getStateFlow("scrollPosition", 0)

    fun toggleSelection(id: String) {
        val current = savedStateHandle.get<Set<String>>("selectedIds") ?: emptySet()
        savedStateHandle["selectedIds"] = if (id in current) current - id else current + id
    }

    fun saveScrollPosition(position: Int) {
        savedStateHandle["scrollPosition"] = position
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SavedStateHandle` | プロセスキル対応状態 |
| `getStateFlow` | Flow形式で取得 |
| `savedStateHandle["key"]` | 値の保存 |
| Navigation引数 | 自動注入 |

- `SavedStateHandle`でプロセスキル後も状態を復元
- `getStateFlow()`でリアクティブに値を監視
- Navigation引数は自動的にSavedStateHandleに注入
- Bundleに保存可能な型（String、Int、Parcelable等）が対象

---

8種類のAndroidアプリテンプレート（SavedState対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ProcessDeath](https://zenn.dev/myougatheaxo/articles/android-compose-compose-process-death-2026)
- [Compose RememberSaveable](https://zenn.dev/myougatheaxo/articles/android-compose-compose-remember-saveable-2026)
- [Hilt ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-viewmodel-2026)
