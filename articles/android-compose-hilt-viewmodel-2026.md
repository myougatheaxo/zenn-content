---
title: "Hilt ViewModel完全ガイド — @HiltViewModel/SavedStateHandle/Assistedインジェクション"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt ViewModel**（@HiltViewModel、SavedStateHandle、AssistedInject、ViewModel Factory）を解説します。

---

## 基本@HiltViewModel

```kotlin
@HiltViewModel
class TaskViewModel @Inject constructor(
    private val taskRepository: TaskRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val _tasks = MutableStateFlow<List<Task>>(emptyList())
    val tasks: StateFlow<List<Task>> = _tasks.asStateFlow()

    val searchQuery = savedStateHandle.getStateFlow("query", "")

    init {
        viewModelScope.launch {
            taskRepository.getTasks().collect { _tasks.value = it }
        }
    }

    fun updateQuery(query: String) {
        savedStateHandle["query"] = query
    }

    fun addTask(title: String) {
        viewModelScope.launch {
            taskRepository.insert(Task(title = title))
        }
    }
}

@Composable
fun TaskScreen(viewModel: TaskViewModel = hiltViewModel()) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    val query by viewModel.searchQuery.collectAsStateWithLifecycle()

    Column {
        TextField(value = query, onValueChange = viewModel::updateQuery)
        LazyColumn {
            items(tasks) { task -> Text(task.title, Modifier.padding(16.dp)) }
        }
    }
}
```

---

## Navigation引数付きViewModel

```kotlin
@HiltViewModel
class DetailViewModel @Inject constructor(
    private val repository: ItemRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val itemId: Long = checkNotNull(savedStateHandle["itemId"])

    val item: StateFlow<Item?> = repository.getById(itemId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}

// NavGraph
composable("detail/{itemId}", arguments = listOf(navArgument("itemId") { type = NavType.LongType })) {
    DetailScreen()
}

@Composable
fun DetailScreen(viewModel: DetailViewModel = hiltViewModel()) {
    val item by viewModel.item.collectAsStateWithLifecycle()
    item?.let { Text(it.name) }
}
```

---

## スコープ付きViewModel

```kotlin
// Activity共有ViewModel
@Composable
fun ScreenA(sharedViewModel: SharedViewModel = hiltViewModel(LocalContext.current as ComponentActivity)) {
    val data by sharedViewModel.data.collectAsStateWithLifecycle()
    Text("Screen A: $data")
}

// NavBackStackEntry共有
@Composable
fun NestedScreen(navController: NavController) {
    val parentEntry = remember(navController) {
        navController.getBackStackEntry("parent_route")
    }
    val parentViewModel: ParentViewModel = hiltViewModel(parentEntry)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@HiltViewModel` | ViewModel自動注入 |
| `hiltViewModel()` | Compose取得 |
| `SavedStateHandle` | プロセス復元対応 |
| Activity共有 | `hiltViewModel(activity)` |

- `@HiltViewModel`+`@Inject constructor`でDI自動化
- `SavedStateHandle`でプロセス終了/復元に対応
- Navigation引数は`SavedStateHandle`から自動取得
- Activity/NavBackStackEntry単位でスコープ指定可能

---

8種類のAndroidアプリテンプレート（Hilt DI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt Module](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-module-2026)
- [Hilt Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-navigation-2026)
- [Hilt Worker](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-worker-2026)
