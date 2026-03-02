---
title: "Compose ProcessDeath完全ガイド — プロセス終了対応/SavedStateHandle/状態復元"
emoji: "💀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lifecycle"]
published: true
---

## この記事で学べること

**Compose ProcessDeath対策**（SavedStateHandle、rememberSaveable、状態復元戦略）を解説します。

---

## SavedStateHandle

```kotlin
@HiltViewModel
class FormViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    var title by mutableStateOf(savedStateHandle.get<String>("title") ?: "")
        private set

    var description by mutableStateOf(savedStateHandle.get<String>("description") ?: "")
        private set

    var selectedCategory by mutableStateOf(savedStateHandle.get<String>("category") ?: "")
        private set

    fun updateTitle(value: String) {
        title = value
        savedStateHandle["title"] = value
    }

    fun updateDescription(value: String) {
        description = value
        savedStateHandle["description"] = value
    }

    fun updateCategory(value: String) {
        selectedCategory = value
        savedStateHandle["category"] = value
    }
}
```

---

## rememberSaveable

```kotlin
@Composable
fun SaveableDemo() {
    // プリミティブ型
    var count by rememberSaveable { mutableIntStateOf(0) }
    var text by rememberSaveable { mutableStateOf("") }

    // カスタムオブジェクト（Saver）
    var filter by rememberSaveable(stateSaver = FilterSaver) {
        mutableStateOf(Filter())
    }

    Column(Modifier.padding(16.dp)) {
        Text("カウント: $count")
        Button(onClick = { count++ }) { Text("増加") }
        TextField(value = text, onValueChange = { text = it })
    }
}

data class Filter(val query: String = "", val sortAsc: Boolean = true)

val FilterSaver = run {
    val queryKey = "query"
    val sortKey = "sort"
    mapSaver(
        save = { mapOf(queryKey to it.query, sortKey to it.sortAsc) },
        restore = { Filter(query = it[queryKey] as String, sortAsc = it[sortKey] as Boolean) }
    )
}
```

---

## StateFlow + SavedStateHandle

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: SearchRepository
) : ViewModel() {

    // SavedStateHandleからStateFlowを直接生成
    val query = savedStateHandle.getStateFlow("query", "")

    val results = query
        .debounce(300)
        .flatMapLatest { q ->
            if (q.isBlank()) flowOf(emptyList())
            else repository.search(q)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun updateQuery(value: String) {
        savedStateHandle["query"] = value
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SavedStateHandle` | ViewModel状態復元 |
| `rememberSaveable` | Composable状態復元 |
| `mapSaver` | カスタム型の保存 |
| `getStateFlow` | SavedState→Flow変換 |

- `SavedStateHandle`でプロセス終了後も状態復元
- `rememberSaveable`でComposableレベルの状態保存
- カスタム型は`Saver`で保存/復元ロジックを定義
- `getStateFlow`でリアクティブな状態監視

---

8種類のAndroidアプリテンプレート（プロセス終了対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose RememberSaveable](https://zenn.dev/myougatheaxo/articles/android-compose-compose-remember-saveable-2026)
- [Compose Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-2026)
- [Hilt ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-viewmodel-2026)
