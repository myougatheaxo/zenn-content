---
title: "Androidエラーハンドリング設計パターン — クラッシュしないアプリを作る"
emoji: "🛠️"
type: "tech"
topics: ["android", "kotlin", "architecture", "errorhandling"]
published: true
---

## この記事で学べること

ネットワークエラー、DB例外、バリデーション失敗。**全てのエラーを適切に処理**してクラッシュしないアプリを作る設計パターンを解説します。

---

## Result型パターン

```kotlin
sealed class AppResult<out T> {
    data class Success<T>(val data: T) : AppResult<T>()
    data class Error(val exception: AppException) : AppResult<Nothing>()
}

sealed class AppException(message: String) : Exception(message) {
    class Network(message: String = "ネットワークエラー") : AppException(message)
    class NotFound(message: String = "データが見つかりません") : AppException(message)
    class Server(message: String = "サーバーエラー") : AppException(message)
    class Unknown(message: String = "予期しないエラー") : AppException(message)
}
```

---

## Repositoryでのエラー変換

```kotlin
class TaskRepository(private val api: TaskApi, private val dao: TaskDao) {

    suspend fun getTasks(): AppResult<List<Task>> {
        return try {
            val tasks = api.fetchTasks()
            dao.insertAll(tasks.map { it.toEntity() })
            AppResult.Success(tasks)
        } catch (e: IOException) {
            // オフライン → ローカルDBから取得
            val cached = dao.getAllTasks()
            if (cached.isNotEmpty()) {
                AppResult.Success(cached.map { it.toTask() })
            } else {
                AppResult.Error(AppException.Network())
            }
        } catch (e: HttpException) {
            when (e.code()) {
                404 -> AppResult.Error(AppException.NotFound())
                in 500..599 -> AppResult.Error(AppException.Server())
                else -> AppResult.Error(AppException.Unknown(e.message()))
            }
        }
    }
}
```

---

## ViewModelでの状態管理

```kotlin
class TaskViewModel(private val repository: TaskRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<TaskUiState>(TaskUiState.Loading)
    val uiState: StateFlow<TaskUiState> = _uiState.asStateFlow()

    fun loadTasks() {
        viewModelScope.launch {
            _uiState.value = TaskUiState.Loading

            when (val result = repository.getTasks()) {
                is AppResult.Success -> {
                    _uiState.value = TaskUiState.Success(result.data)
                }
                is AppResult.Error -> {
                    _uiState.value = TaskUiState.Error(
                        result.exception.message ?: "エラー"
                    )
                }
            }
        }
    }
}

sealed class TaskUiState {
    data object Loading : TaskUiState()
    data class Success(val tasks: List<Task>) : TaskUiState()
    data class Error(val message: String) : TaskUiState()
}
```

---

## Composeでのエラー表示

```kotlin
@Composable
fun TaskScreen(viewModel: TaskViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is TaskUiState.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is TaskUiState.Success -> {
            val tasks = (uiState as TaskUiState.Success).tasks
            LazyColumn {
                items(tasks, key = { it.id }) { task ->
                    TaskItem(task)
                }
            }
        }
        is TaskUiState.Error -> {
            ErrorScreen(
                message = (uiState as TaskUiState.Error).message,
                onRetry = { viewModel.loadTasks() }
            )
        }
    }
}

@Composable
fun ErrorScreen(message: String, onRetry: () -> Unit) {
    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(Icons.Default.Warning, null, Modifier.size(64.dp))
        Spacer(Modifier.height(16.dp))
        Text(message, style = MaterialTheme.typography.bodyLarge)
        Spacer(Modifier.height(16.dp))
        Button(onClick = onRetry) { Text("再試行") }
    }
}
```

---

## Flowでのエラーハンドリング

```kotlin
val tasks: StateFlow<TaskUiState> = repository.observeTasks()
    .map<List<Task>, TaskUiState> { TaskUiState.Success(it) }
    .catch { e ->
        emit(TaskUiState.Error(e.message ?: "エラー"))
    }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), TaskUiState.Loading)
```

---

## バリデーションエラー

```kotlin
data class ValidationResult(
    val isValid: Boolean,
    val errorMessage: String? = null
)

fun validateEmail(email: String): ValidationResult {
    return when {
        email.isBlank() -> ValidationResult(false, "メールアドレスを入力してください")
        !email.contains("@") -> ValidationResult(false, "正しいメールアドレスを入力してください")
        else -> ValidationResult(true)
    }
}
```

---

## まとめ

| レイヤー | エラー処理 |
|---------|-----------|
| Repository | try-catch → Result型に変換 |
| ViewModel | Result → UiState に変換 |
| UI | UiState.Error → エラー画面表示 |
| Flow | .catch { } でエラーを捕捉 |
| 入力 | ValidationResultで検証 |

---

8種類のAndroidアプリテンプレート（エラーハンドリング設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin sealed class完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
