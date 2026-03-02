---
title: "Compose状態管理完全ガイド — remember/State/StateFlow/UiState"
emoji: "🧠"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

Composeの状態管理を、`remember` / `mutableStateOf` / `StateFlow` / `UiState`パターンまで体系的に解説します。

---

## remember + mutableStateOf

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("カウント: $count")
    }
}
```

- `remember`: Recomposition間で値を保持
- `mutableStateOf`: 変更時にRecompositionをトリガー

---

## rememberSaveable（画面回転対応）

```kotlin
@Composable
fun SearchBar() {
    var query by rememberSaveable { mutableStateOf("") }

    OutlinedTextField(
        value = query,
        onValueChange = { query = it },
        label = { Text("検索") }
    )
}
```

`rememberSaveable`はプロセス再生成後も値を復元。

---

## State Hoisting（状態の引き上げ）

```kotlin
// ❌ 状態を内部に持つ
@Composable
fun BadToggle() {
    var checked by remember { mutableStateOf(false) }
    Switch(checked = checked, onCheckedChange = { checked = it })
}

// ✅ 状態を引き上げる
@Composable
fun GoodToggle(checked: Boolean, onCheckedChange: (Boolean) -> Unit) {
    Switch(checked = checked, onCheckedChange = onCheckedChange)
}

// 使う側
@Composable
fun SettingsScreen() {
    var darkMode by remember { mutableStateOf(false) }
    GoodToggle(checked = darkMode, onCheckedChange = { darkMode = it })
}
```

---

## ViewModel + StateFlow

```kotlin
class TaskViewModel : ViewModel() {
    private val _tasks = MutableStateFlow<List<Task>>(emptyList())
    val tasks: StateFlow<List<Task>> = _tasks.asStateFlow()

    fun addTask(title: String) {
        _tasks.update { current ->
            current + Task(id = current.size + 1, title = title)
        }
    }

    fun toggleTask(id: Int) {
        _tasks.update { current ->
            current.map {
                if (it.id == id) it.copy(isCompleted = !it.isCompleted) else it
            }
        }
    }
}

@Composable
fun TaskScreen(viewModel: TaskViewModel) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()

    LazyColumn {
        items(tasks, key = { it.id }) { task ->
            TaskItem(
                task = task,
                onToggle = { viewModel.toggleTask(task.id) }
            )
        }
    }
}
```

---

## UiStateパターン

```kotlin
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

class ArticleViewModel(private val repository: ArticleRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Article>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Article>>> = _uiState.asStateFlow()

    init {
        loadArticles()
    }

    private fun loadArticles() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val articles = repository.getArticles()
                _uiState.value = UiState.Success(articles)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "エラーが発生しました")
            }
        }
    }

    fun retry() = loadArticles()
}
```

---

## UiState対応UI

```kotlin
@Composable
fun ArticleScreen(viewModel: ArticleViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UiState.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is UiState.Success -> {
            LazyColumn {
                items(state.data, key = { it.id }) { article ->
                    ArticleItem(article)
                }
            }
        }
        is UiState.Error -> {
            Column(
                Modifier.fillMaxSize(),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text(state.message, color = MaterialTheme.colorScheme.error)
                Spacer(Modifier.height(16.dp))
                Button(onClick = { viewModel.retry() }) {
                    Text("再試行")
                }
            }
        }
    }
}
```

---

## derivedStateOf（派生状態）

```kotlin
@Composable
fun FilteredList(items: List<Item>) {
    var query by remember { mutableStateOf("") }

    val filteredItems by remember(items) {
        derivedStateOf {
            if (query.isBlank()) items
            else items.filter { it.title.contains(query, ignoreCase = true) }
        }
    }

    Column {
        TextField(value = query, onValueChange = { query = it })
        LazyColumn {
            items(filteredItems) { item -> Text(item.title) }
        }
    }
}
```

---

## 使い分けまとめ

| 場面 | 方法 |
|------|------|
| UI内の一時状態 | `remember { mutableStateOf() }` |
| 画面回転対応 | `rememberSaveable` |
| ビジネスロジック | ViewModel + `StateFlow` |
| 非同期データ取得 | `UiState` sealed class |
| 派生計算 | `derivedStateOf` |
| リスト操作 | `MutableStateFlow` + `update` |

---

8種類のAndroidアプリテンプレート（状態管理ベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel完全ガイド](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Kotlin Coroutines & Flow入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-2026)
- [Side Effects完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-side-effects-2026)
