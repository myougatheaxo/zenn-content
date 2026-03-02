---
title: "検索/フィルタリングガイド — SearchBar + Flow debounce実装"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "search"]
published: true
---

## この記事で学べること

Composeでの**検索/フィルタリング**（SearchBar、debounce、リアルタイムフィルター）を解説します。

---

## Material3 SearchBar

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel = viewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val results by viewModel.results.collectAsStateWithLifecycle()
    val suggestions by viewModel.suggestions.collectAsStateWithLifecycle()
    var active by remember { mutableStateOf(false) }

    SearchBar(
        query = query,
        onQueryChange = { viewModel.updateQuery(it) },
        onSearch = {
            active = false
            viewModel.search(it)
        },
        active = active,
        onActiveChange = { active = it },
        placeholder = { Text("検索...") },
        leadingIcon = { Icon(Icons.Default.Search, null) },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = { viewModel.updateQuery("") }) {
                    Icon(Icons.Default.Clear, "クリア")
                }
            }
        },
        modifier = Modifier.fillMaxWidth()
    ) {
        // 検索候補
        suggestions.forEach { suggestion ->
            ListItem(
                headlineContent = { Text(suggestion) },
                leadingContent = { Icon(Icons.Default.History, null) },
                modifier = Modifier.clickable {
                    viewModel.updateQuery(suggestion)
                    active = false
                    viewModel.search(suggestion)
                }
            )
        }
    }

    // 検索結果
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
class SearchViewModel(private val repository: ItemRepository) : ViewModel() {

    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    // debounce: 入力停止300ms後に検索
    val results: StateFlow<List<Item>> = _query
        .debounce(300)
        .distinctUntilChanged()
        .flatMapLatest { q ->
            if (q.isBlank()) flowOf(emptyList())
            else repository.search(q)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    val suggestions: StateFlow<List<String>> = _query
        .debounce(200)
        .map { q ->
            if (q.length < 2) emptyList()
            else repository.getSuggestions(q)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun updateQuery(query: String) {
        _query.value = query
    }

    fun search(query: String) {
        _query.value = query
    }
}
```

---

## チップフィルター

```kotlin
@Composable
fun FilterableList(viewModel: ItemViewModel = viewModel()) {
    val items by viewModel.filteredItems.collectAsStateWithLifecycle()
    val categories by viewModel.categories.collectAsStateWithLifecycle()
    val selectedCategory by viewModel.selectedCategory.collectAsStateWithLifecycle()

    Column {
        LazyRow(
            contentPadding = PaddingValues(horizontal = 16.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            item {
                FilterChip(
                    selected = selectedCategory == null,
                    onClick = { viewModel.selectCategory(null) },
                    label = { Text("すべて") }
                )
            }
            items(categories) { category ->
                FilterChip(
                    selected = selectedCategory == category,
                    onClick = { viewModel.selectCategory(category) },
                    label = { Text(category) }
                )
            }
        }

        LazyColumn {
            items(items, key = { it.id }) { item ->
                ListItem(
                    headlineContent = { Text(item.title) },
                    supportingContent = { Text(item.category) }
                )
            }
        }
    }
}
```

---

## 複合フィルター

```kotlin
class ItemViewModel(private val repository: ItemRepository) : ViewModel() {

    private val _query = MutableStateFlow("")
    private val _selectedCategory = MutableStateFlow<String?>(null)
    private val _sortOrder = MutableStateFlow(SortOrder.NEWEST)

    val filteredItems: StateFlow<List<Item>> = combine(
        _query.debounce(300),
        _selectedCategory,
        _sortOrder,
        repository.getAllItems()
    ) { query, category, sort, items ->
        items
            .filter { item ->
                (query.isBlank() || item.title.contains(query, ignoreCase = true)) &&
                (category == null || item.category == category)
            }
            .let { filtered ->
                when (sort) {
                    SortOrder.NEWEST -> filtered.sortedByDescending { it.createdAt }
                    SortOrder.OLDEST -> filtered.sortedBy { it.createdAt }
                    SortOrder.NAME -> filtered.sortedBy { it.title }
                }
            }
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun updateQuery(query: String) { _query.value = query }
    fun selectCategory(category: String?) { _selectedCategory.value = category }
    fun setSortOrder(order: SortOrder) { _sortOrder.value = order }
}
```

---

## まとめ

- Material3 `SearchBar`でフル機能の検索UI
- `debounce(300)`で入力中の無駄なAPIコール防止
- `distinctUntilChanged()`で同じクエリの重複防止
- `FilterChip`でカテゴリフィルター
- `combine`で複数条件の複合フィルター
- `flatMapLatest`で最新の検索のみ実行

---

8種類のAndroidアプリテンプレート（検索/フィルター設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [SearchBar実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-search-bar-2026)
- [Flow + Lifecycleガイド](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
