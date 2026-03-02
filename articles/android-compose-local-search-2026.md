---
title: "ローカル検索/フィルター実装ガイド — SearchBar+Flow"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "search"]
published: true
---

## この記事で学べること

Composeでの**ローカル検索/フィルター**（SearchBar、debounce、複合フィルター、検索履歴）を解説します。

---

## Material3 SearchBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    var query by remember { mutableStateOf("") }
    var active by remember { mutableStateOf(false) }
    val results by viewModel.searchResults.collectAsStateWithLifecycle()
    val suggestions by viewModel.suggestions.collectAsStateWithLifecycle()

    SearchBar(
        query = query,
        onQueryChange = {
            query = it
            viewModel.search(it)
        },
        onSearch = {
            active = false
            viewModel.addToHistory(it)
        },
        active = active,
        onActiveChange = { active = it },
        placeholder = { Text("検索...") },
        leadingIcon = { Icon(Icons.Default.Search, null) },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = { query = ""; viewModel.search("") }) {
                    Icon(Icons.Default.Clear, "クリア")
                }
            }
        },
        modifier = Modifier.fillMaxWidth()
    ) {
        // サジェスション
        suggestions.forEach { suggestion ->
            ListItem(
                headlineContent = { Text(suggestion) },
                leadingContent = { Icon(Icons.Default.History, null) },
                modifier = Modifier.clickable {
                    query = suggestion
                    viewModel.search(suggestion)
                    active = false
                }
            )
        }
    }

    // 結果リスト
    LazyColumn {
        items(results, key = { it.id }) { item ->
            ResultItem(item)
        }
    }
}
```

---

## ViewModel（debounce付き）

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {

    private val _query = MutableStateFlow("")

    val searchResults: StateFlow<List<Item>> = _query
        .debounce(300) // 300ms待って検索
        .flatMapLatest { query ->
            if (query.isBlank()) {
                repository.observeAll()
            } else {
                repository.search(query)
            }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    private val _history = MutableStateFlow<List<String>>(emptyList())
    val suggestions: StateFlow<List<String>> = combine(_query, _history) { query, history ->
        if (query.isBlank()) history.take(5)
        else history.filter { it.contains(query, ignoreCase = true) }.take(5)
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun search(query: String) {
        _query.value = query
    }

    fun addToHistory(query: String) {
        if (query.isNotBlank()) {
            _history.update { (listOf(query) + it).distinct().take(20) }
        }
    }
}
```

---

## 複合フィルター

```kotlin
data class FilterState(
    val query: String = "",
    val category: String? = null,
    val sortBy: SortBy = SortBy.DATE_DESC,
    val priceRange: ClosedFloatingPointRange<Float> = 0f..10000f
)

enum class SortBy { DATE_DESC, DATE_ASC, PRICE_ASC, PRICE_DESC, NAME }

@HiltViewModel
class FilterViewModel @Inject constructor(
    private val repository: ProductRepository
) : ViewModel() {

    private val _filter = MutableStateFlow(FilterState())
    val filter: StateFlow<FilterState> = _filter.asStateFlow()

    val filteredProducts: StateFlow<List<Product>> = _filter
        .debounce(200)
        .flatMapLatest { filter ->
            repository.observeAll().map { products ->
                products
                    .filter { product ->
                        (filter.query.isBlank() ||
                            product.name.contains(filter.query, ignoreCase = true)) &&
                        (filter.category == null || product.category == filter.category) &&
                        product.price in filter.priceRange
                    }
                    .sortedWith(when (filter.sortBy) {
                        SortBy.DATE_DESC -> compareByDescending { it.createdAt }
                        SortBy.DATE_ASC -> compareBy { it.createdAt }
                        SortBy.PRICE_ASC -> compareBy { it.price }
                        SortBy.PRICE_DESC -> compareByDescending { it.price }
                        SortBy.NAME -> compareBy { it.name }
                    })
            }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun updateQuery(query: String) { _filter.update { it.copy(query = query) } }
    fun setCategory(category: String?) { _filter.update { it.copy(category = category) } }
    fun setSortBy(sortBy: SortBy) { _filter.update { it.copy(sortBy = sortBy) } }
    fun resetFilters() { _filter.value = FilterState() }
}
```

---

## フィルターチップUI

```kotlin
@Composable
fun FilterChips(
    categories: List<String>,
    selectedCategory: String?,
    onSelect: (String?) -> Unit
) {
    LazyRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        contentPadding = PaddingValues(horizontal = 16.dp)
    ) {
        item {
            FilterChip(
                selected = selectedCategory == null,
                onClick = { onSelect(null) },
                label = { Text("すべて") }
            )
        }
        items(categories) { category ->
            FilterChip(
                selected = selectedCategory == category,
                onClick = { onSelect(category) },
                label = { Text(category) }
            )
        }
    }
}
```

---

## まとめ

- `SearchBar`でMaterial3準拠の検索UI
- `debounce(300)`で入力中の無駄な検索を防止
- `flatMapLatest`で最新クエリのみ処理
- `combine`で複合フィルター条件を統合
- `FilterChip`でカテゴリフィルター
- 検索履歴は`MutableStateFlow<List<String>>`で管理

---

8種類のAndroidアプリテンプレート（検索/フィルター実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow演算子](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-operators-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-tips-2026)
- [Chip/FilterChip](https://zenn.dev/myougatheaxo/articles/android-compose-chip-selection-2026)
