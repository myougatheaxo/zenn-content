---
title: "Compose State管理完全ガイド — remember/rememberSaveable/StateFlow/状態ホイスティング"
emoji: "🧠"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "compose"]
published: true
---

## この記事で学べること

**Compose State管理**（remember、rememberSaveable、StateFlow、状態ホイスティング、UiState設計）を解説します。

---

## remember / rememberSaveable

```kotlin
@Composable
fun CounterExample() {
    // プロセスデスでリセット
    var count by remember { mutableIntStateOf(0) }

    // プロセスデスでも保持
    var savedCount by rememberSaveable { mutableIntStateOf(0) }

    // カスタムSaver
    var user by rememberSaveable(stateSaver = UserSaver) {
        mutableStateOf(User("", ""))
    }

    Column {
        Text("count: $count")
        Text("saved: $savedCount")
        Button(onClick = { count++; savedCount++ }) { Text("+1") }
    }
}

val UserSaver = run {
    val nameKey = "name"
    val emailKey = "email"
    mapSaver(
        save = { mapOf(nameKey to it.name, emailKey to it.email) },
        restore = { User(it[nameKey] as String, it[emailKey] as String) }
    )
}
```

---

## UiState設計

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

class ItemViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Item>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Item>>> = _uiState.asStateFlow()

    init { loadItems() }

    private fun loadItems() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val items = repository.getItems()
                _uiState.value = UiState.Success(items)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }

    fun retry() = loadItems()
}
```

---

## 状態ホイスティング

```kotlin
// Statelessコンポーネント（推奨）
@Composable
fun TodoItem(
    todo: Todo,
    onToggle: (Boolean) -> Unit,
    onDelete: () -> Unit
) {
    Row(
        Modifier.fillMaxWidth().padding(8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Checkbox(checked = todo.completed, onCheckedChange = onToggle)
        Text(todo.title, Modifier.weight(1f))
        IconButton(onClick = onDelete) { Icon(Icons.Default.Delete, null) }
    }
}

// Statefulコンテナ
@Composable
fun TodoScreen(viewModel: TodoViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Error -> ErrorMessage(state.message, onRetry = viewModel::retry)
        is UiState.Success -> {
            LazyColumn {
                items(state.data, key = { it.id }) { todo ->
                    TodoItem(
                        todo = todo,
                        onToggle = { viewModel.toggleTodo(todo.id) },
                        onDelete = { viewModel.deleteTodo(todo.id) }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

| 手法 | 保持範囲 |
|------|---------|
| `remember` | 再コンポーズまで |
| `rememberSaveable` | 画面回転/プロセスデス |
| `ViewModel StateFlow` | ViewModelスコープ |
| `SavedStateHandle` | プロセスデス |

- `remember`は一時的な状態に使用
- `rememberSaveable`で画面回転対応
- `StateFlow`+`collectAsStateWithLifecycle`でUI状態管理
- 状態ホイスティングでStatelessコンポーネント設計

---

8種類のAndroidアプリテンプレート（State管理設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [副作用API](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effects-2026)
- [再コンポーズ](https://zenn.dev/myougatheaxo/articles/android-compose-compose-recomposition-2026)
- [ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-testing-2026)
