---
title: "Compose状態管理 上級編 — MVI/Redux/Orbit"
emoji: "🔧"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "mvi"]
published: true
---

## この記事で学べること

Composeの上級**状態管理**パターン（MVI、Reducer、Side Effect、Orbitライブラリ）を解説します。

---

## MVI（Model-View-Intent）

```kotlin
// State
data class CounterState(
    val count: Int = 0,
    val isLoading: Boolean = false
)

// Intent（ユーザーアクション）
sealed interface CounterIntent {
    data object Increment : CounterIntent
    data object Decrement : CounterIntent
    data object Reset : CounterIntent
    data class SetValue(val value: Int) : CounterIntent
}

// ViewModel
@HiltViewModel
class CounterViewModel @Inject constructor() : ViewModel() {

    private val _state = MutableStateFlow(CounterState())
    val state: StateFlow<CounterState> = _state.asStateFlow()

    fun processIntent(intent: CounterIntent) {
        when (intent) {
            CounterIntent.Increment -> _state.update { it.copy(count = it.count + 1) }
            CounterIntent.Decrement -> _state.update { it.copy(count = it.count - 1) }
            CounterIntent.Reset -> _state.update { it.copy(count = 0) }
            is CounterIntent.SetValue -> _state.update { it.copy(count = intent.value) }
        }
    }
}

// Compose
@Composable
fun CounterScreen(viewModel: CounterViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text("Count: ${state.count}", style = MaterialTheme.typography.displayLarge)
        Row {
            Button(onClick = { viewModel.processIntent(CounterIntent.Decrement) }) {
                Text("-")
            }
            Button(onClick = { viewModel.processIntent(CounterIntent.Increment) }) {
                Text("+")
            }
        }
    }
}
```

---

## Reducer パターン

```kotlin
// 複雑な状態遷移をReducerで一元管理
data class TodoState(
    val items: List<Todo> = emptyList(),
    val filter: TodoFilter = TodoFilter.ALL,
    val isLoading: Boolean = false,
    val error: String? = null
)

sealed interface TodoAction {
    data object LoadTodos : TodoAction
    data class TodosLoaded(val items: List<Todo>) : TodoAction
    data class LoadError(val message: String) : TodoAction
    data class AddTodo(val title: String) : TodoAction
    data class ToggleTodo(val id: String) : TodoAction
    data class DeleteTodo(val id: String) : TodoAction
    data class SetFilter(val filter: TodoFilter) : TodoAction
}

fun todoReducer(state: TodoState, action: TodoAction): TodoState = when (action) {
    TodoAction.LoadTodos -> state.copy(isLoading = true, error = null)
    is TodoAction.TodosLoaded -> state.copy(items = action.items, isLoading = false)
    is TodoAction.LoadError -> state.copy(error = action.message, isLoading = false)
    is TodoAction.AddTodo -> state.copy(
        items = state.items + Todo(id = UUID.randomUUID().toString(), title = action.title)
    )
    is TodoAction.ToggleTodo -> state.copy(
        items = state.items.map {
            if (it.id == action.id) it.copy(completed = !it.completed) else it
        }
    )
    is TodoAction.DeleteTodo -> state.copy(
        items = state.items.filter { it.id != action.id }
    )
    is TodoAction.SetFilter -> state.copy(filter = action.filter)
}

@HiltViewModel
class TodoViewModel @Inject constructor(
    private val repository: TodoRepository
) : ViewModel() {

    private val _state = MutableStateFlow(TodoState())
    val state: StateFlow<TodoState> = _state.asStateFlow()

    fun dispatch(action: TodoAction) {
        _state.update { todoReducer(it, action) }

        // Side Effects
        when (action) {
            TodoAction.LoadTodos -> {
                viewModelScope.launch {
                    try {
                        val todos = repository.getAll()
                        dispatch(TodoAction.TodosLoaded(todos))
                    } catch (e: Exception) {
                        dispatch(TodoAction.LoadError(e.message ?: "Error"))
                    }
                }
            }
            else -> {}
        }
    }
}
```

---

## Side Effect Channel

```kotlin
sealed interface UiEffect {
    data class ShowSnackbar(val message: String) : UiEffect
    data class Navigate(val route: String) : UiEffect
    data object ScrollToTop : UiEffect
}

@HiltViewModel
class FormViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _state = MutableStateFlow(FormState())
    val state: StateFlow<FormState> = _state.asStateFlow()

    private val _effects = Channel<UiEffect>(Channel.BUFFERED)
    val effects = _effects.receiveAsFlow()

    fun onSubmit() {
        viewModelScope.launch {
            _state.update { it.copy(isSubmitting = true) }
            try {
                repository.submit(state.value.toRequest())
                _effects.send(UiEffect.ShowSnackbar("保存しました"))
                _effects.send(UiEffect.Navigate("home"))
            } catch (e: Exception) {
                _effects.send(UiEffect.ShowSnackbar("エラー: ${e.message}"))
            } finally {
                _state.update { it.copy(isSubmitting = false) }
            }
        }
    }
}

// Compose
@Composable
fun FormScreen(
    viewModel: FormViewModel = hiltViewModel(),
    navController: NavController,
    snackbarHostState: SnackbarHostState
) {
    LaunchedEffect(Unit) {
        viewModel.effects.collect { effect ->
            when (effect) {
                is UiEffect.ShowSnackbar -> snackbarHostState.showSnackbar(effect.message)
                is UiEffect.Navigate -> navController.navigate(effect.route)
                UiEffect.ScrollToTop -> { /* scroll */ }
            }
        }
    }
}
```

---

## 複合状態の最適化

```kotlin
// ❌ 1つの巨大なStateだとすべてリコンポーズ
data class ScreenState(
    val users: List<User>,
    val searchQuery: String,
    val selectedTab: Int,
    val isLoading: Boolean
)

// ✅ 独立した状態を分割
@HiltViewModel
class OptimizedViewModel @Inject constructor() : ViewModel() {
    val users = MutableStateFlow<List<User>>(emptyList())
    val searchQuery = MutableStateFlow("")
    val selectedTab = MutableStateFlow(0)
    val isLoading = MutableStateFlow(false)

    // 派生状態
    val filteredUsers: StateFlow<List<User>> = combine(users, searchQuery) { list, query ->
        if (query.isBlank()) list
        else list.filter { it.name.contains(query, ignoreCase = true) }
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}
```

---

## まとめ

| パターン | 複雑度 | 用途 |
|---------|--------|------|
| StateFlow直接更新 | 低 | 単純な画面 |
| MVI (Intent) | 中 | 中規模アプリ |
| Reducer | 高 | 複雑な状態遷移 |
| Side Effect Channel | - | 1回限りイベント |

- `sealed interface`でIntentを定義→`when`で分岐
- Reducerで純粋関数的な状態遷移
- `Channel<UiEffect>`で1回限りのUIイベント
- 状態分割で不要なリコンポーズ防止

---

8種類のAndroidアプリテンプレート（MVI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + Hilt](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
- [Result型パターン](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-result-2026)
- [remember/derivedStateOf](https://zenn.dev/myougatheaxo/articles/android-compose-remember-derived-2026)
