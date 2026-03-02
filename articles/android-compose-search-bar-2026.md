---
title: "Compose SearchBar実装ガイド — Material3の検索UIを作る"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**SearchBar**コンポーネントを使った検索UIの実装方法を解説します。リアルタイム検索、検索履歴、サジェストも含みます。

---

## 基本のSearchBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BasicSearchBar() {
    var query by rememberSaveable { mutableStateOf("") }
    var active by rememberSaveable { mutableStateOf(false) }

    SearchBar(
        query = query,
        onQueryChange = { query = it },
        onSearch = {
            active = false
            // 検索実行
        },
        active = active,
        onActiveChange = { active = it },
        placeholder = { Text("検索...") },
        leadingIcon = { Icon(Icons.Default.Search, "検索") },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = { query = "" }) {
                    Icon(Icons.Default.Clear, "クリア")
                }
            }
        }
    ) {
        // 検索サジェスト・履歴
    }
}
```

---

## 検索サジェスト

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SearchWithSuggestions(viewModel: SearchViewModel) {
    var query by rememberSaveable { mutableStateOf("") }
    var active by rememberSaveable { mutableStateOf(false) }
    val suggestions by viewModel.suggestions.collectAsStateWithLifecycle()

    SearchBar(
        query = query,
        onQueryChange = {
            query = it
            viewModel.onQueryChanged(it)
        },
        onSearch = {
            active = false
            viewModel.search(query)
        },
        active = active,
        onActiveChange = { active = it },
        placeholder = { Text("検索...") },
        leadingIcon = { Icon(Icons.Default.Search, null) }
    ) {
        suggestions.forEach { suggestion ->
            ListItem(
                headlineContent = { Text(suggestion) },
                leadingContent = { Icon(Icons.Default.History, null) },
                modifier = Modifier.clickable {
                    query = suggestion
                    active = false
                    viewModel.search(suggestion)
                }
            )
        }
    }
}
```

---

## ViewModel（debounce付き）

```kotlin
class SearchViewModel(private val repository: SearchRepository) : ViewModel() {
    private val _query = MutableStateFlow("")

    val suggestions: StateFlow<List<String>> = _query
        .debounce(300)
        .filter { it.length >= 2 }
        .flatMapLatest { query ->
            repository.getSuggestions(query)
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    val results: StateFlow<List<Item>> = _query
        .debounce(500)
        .filter { it.length >= 2 }
        .flatMapLatest { query ->
            repository.search(query)
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun onQueryChanged(query: String) {
        _query.value = query
    }

    fun search(query: String) {
        _query.value = query
    }
}
```

---

## 検索結果の表示

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    val results by viewModel.results.collectAsStateWithLifecycle()

    Column {
        SearchWithSuggestions(viewModel)

        LazyColumn {
            items(results, key = { it.id }) { item ->
                ListItem(
                    headlineContent = { Text(item.title) },
                    supportingContent = { Text(item.description) }
                )
            }
        }
    }
}
```

---

## ローカルフィルタリング

```kotlin
@Composable
fun FilterableList(items: List<Item>) {
    var query by rememberSaveable { mutableStateOf("") }

    val filteredItems = remember(items, query) {
        if (query.isBlank()) items
        else items.filter {
            it.title.contains(query, ignoreCase = true)
        }
    }

    Column {
        OutlinedTextField(
            value = query,
            onValueChange = { query = it },
            label = { Text("フィルタ") },
            leadingIcon = { Icon(Icons.Default.Search, null) },
            modifier = Modifier.fillMaxWidth().padding(16.dp)
        )

        LazyColumn {
            items(filteredItems, key = { it.id }) { item ->
                Text(item.title, Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## まとめ

- `SearchBar`でMaterial3準拠の検索UI
- `debounce(300)`でリアルタイム検索のAPI負荷軽減
- `flatMapLatest`で最新クエリのみ実行
- サジェスト・検索履歴をドロップダウン表示
- ローカルフィルタは`remember(items, query)`で最適化

---

8種類のAndroidアプリテンプレート（検索機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [Kotlin Coroutines & Flow入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
