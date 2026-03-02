---
title: "SearchBar完全ガイド — SearchBar/DockedSearchBar/履歴/サジェスト"
emoji: "🔎"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**SearchBar**（SearchBar、DockedSearchBar、検索履歴、サジェスト表示）を解説します。

---

## 基本SearchBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SearchBarExample() {
    var query by rememberSaveable { mutableStateOf("") }
    var active by rememberSaveable { mutableStateOf(false) }
    val history = remember { mutableStateListOf("Kotlin", "Compose", "Room") }

    SearchBar(
        query = query,
        onQueryChange = { query = it },
        onSearch = {
            if (query.isNotBlank() && query !in history) history.add(0, query)
            active = false
        },
        active = active,
        onActiveChange = { active = it },
        placeholder = { Text("検索...") },
        leadingIcon = { Icon(Icons.Default.Search, null) },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = { query = "" }) {
                    Icon(Icons.Default.Clear, "クリア")
                }
            }
        },
        modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp)
    ) {
        history.filter { it.contains(query, ignoreCase = true) }.forEach { item ->
            ListItem(
                headlineContent = { Text(item) },
                leadingContent = { Icon(Icons.Default.History, null) },
                modifier = Modifier.clickable { query = item; active = false }
            )
        }
    }
}
```

---

## DockedSearchBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DockedSearchExample(items: List<String>) {
    var query by remember { mutableStateOf("") }
    var active by remember { mutableStateOf(false) }
    val filtered = items.filter { it.contains(query, ignoreCase = true) }

    Column(Modifier.padding(16.dp)) {
        DockedSearchBar(
            query = query,
            onQueryChange = { query = it },
            onSearch = { active = false },
            active = active,
            onActiveChange = { active = it },
            placeholder = { Text("アイテムを検索") },
            leadingIcon = { Icon(Icons.Default.Search, null) },
            modifier = Modifier.fillMaxWidth()
        ) {
            filtered.take(5).forEach { item ->
                ListItem(
                    headlineContent = { Text(item) },
                    modifier = Modifier.clickable { query = item; active = false }
                )
            }
        }

        Spacer(Modifier.height(16.dp))

        LazyColumn {
            items(filtered) { item ->
                ListItem(headlineContent = { Text(item) })
            }
        }
    }
}
```

---

## ViewModel連携

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    private val _query = MutableStateFlow("")
    val query = _query.asStateFlow()

    val suggestions = _query
        .debounce(300)
        .filter { it.length >= 2 }
        .flatMapLatest { repository.getSuggestions(it) }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    val results = _query
        .debounce(500)
        .filter { it.length >= 2 }
        .flatMapLatest { repository.search(it) }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun updateQuery(q: String) { _query.value = q }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SearchBar` | フルスクリーン検索 |
| `DockedSearchBar` | インライン検索 |
| `active` | 展開状態制御 |
| `onSearch` | 検索実行 |

- `SearchBar`でMaterial3の検索UI
- `DockedSearchBar`で画面内配置の検索
- `debounce`でリアルタイムサジェスト
- 検索履歴をリスト表示

---

8種類のAndroidアプリテンプレート（検索機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room FTS4](https://zenn.dev/myougatheaxo/articles/android-compose-room-fts4-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
- [Chip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-chip-2026)
