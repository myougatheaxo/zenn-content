---
title: "ViewModel SavedState完全ガイド — SavedStateHandle/プロセスキル復元/Navigation引数"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "viewmodel"]
published: true
---

## この記事で学べること

**ViewModel SavedState**（SavedStateHandle、プロセスキル対応、Navigation引数の受け取り）を解説します。

---

## SavedStateHandle基本

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: SearchRepository
) : ViewModel() {
    // プロセスキル後も復元されるStateFlow
    val query = savedStateHandle.getStateFlow("query", "")

    val results = query
        .debounce(300)
        .filter { it.length >= 2 }
        .flatMapLatest { repository.search(it) }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun updateQuery(newQuery: String) {
        savedStateHandle["query"] = newQuery
    }
}
```

---

## Navigation引数の受け取り

```kotlin
@HiltViewModel
class DetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: ItemRepository
) : ViewModel() {
    // Navigation引数を取得
    private val itemId: Long = savedStateHandle.get<Long>("itemId")
        ?: throw IllegalArgumentException("itemId is required")

    val item = repository.getItem(itemId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)

    fun deleteItem() {
        viewModelScope.launch {
            repository.delete(itemId)
        }
    }
}

// NavGraph
composable(
    route = "detail/{itemId}",
    arguments = listOf(navArgument("itemId") { type = NavType.LongType })
) {
    DetailScreen()
}
```

---

## 複数状態の保存

```kotlin
@HiltViewModel
class FilterViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    val selectedCategory = savedStateHandle.getStateFlow("category", "all")
    val sortOrder = savedStateHandle.getStateFlow("sort", "newest")
    val priceMin = savedStateHandle.getStateFlow("priceMin", 0)
    val priceMax = savedStateHandle.getStateFlow("priceMax", 100000)

    fun updateCategory(category: String) { savedStateHandle["category"] = category }
    fun updateSort(sort: String) { savedStateHandle["sort"] = sort }
    fun updatePriceRange(min: Int, max: Int) {
        savedStateHandle["priceMin"] = min
        savedStateHandle["priceMax"] = max
    }

    fun resetFilters() {
        savedStateHandle["category"] = "all"
        savedStateHandle["sort"] = "newest"
        savedStateHandle["priceMin"] = 0
        savedStateHandle["priceMax"] = 100000
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SavedStateHandle` | プロセスキル対応状態 |
| `getStateFlow` | リアクティブ状態取得 |
| `savedStateHandle["key"]` | 値の保存 |
| `get<Type>("key")` | Navigation引数取得 |

- `SavedStateHandle`でプロセスキル後の状態復元
- `getStateFlow`でFlowとして監視
- Navigation引数を`SavedStateHandle`から直接取得
- Hiltと組み合わせて自動注入

---

8種類のAndroidアプリテンプレート（ViewModel実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State Management](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-2026)
- [Hilt/DI](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-2026)
