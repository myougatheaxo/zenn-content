---
title: "Coroutine例外処理完全ガイド — CoroutineExceptionHandler/SupervisorJob/構造化並行性"
emoji: "⚠️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Coroutine例外処理**（CoroutineExceptionHandler、SupervisorJob、構造化並行性、エラーリカバリ）を解説します。

---

## CoroutineExceptionHandler

```kotlin
class TaskViewModel @Inject constructor(
    private val repository: TaskRepository
) : ViewModel() {

    private val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        _uiState.value = UiState.Error(throwable.message ?: "Unknown error")
    }

    private val _uiState = MutableStateFlow<UiState<List<Task>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Task>>> = _uiState.asStateFlow()

    fun loadTasks() {
        viewModelScope.launch(exceptionHandler) {
            _uiState.value = UiState.Loading
            val tasks = repository.getTasks()
            _uiState.value = UiState.Success(tasks)
        }
    }
}
```

---

## SupervisorJob

```kotlin
class BatchViewModel @Inject constructor() : ViewModel() {

    // SupervisorJob: 子の失敗が他の子に伝播しない
    fun executeBatch(items: List<Item>) {
        viewModelScope.launch {
            supervisorScope {
                items.forEach { item ->
                    launch {
                        try {
                            processItem(item)
                        } catch (e: Exception) {
                            // この子の失敗は他の子に影響しない
                            logError("Item ${item.id} failed: ${e.message}")
                        }
                    }
                }
            }
        }
    }

    // 並列実行 + 一部失敗許容
    suspend fun fetchAllData(): DashboardData = coroutineScope {
        val users = async(SupervisorJob()) {
            try { repository.getUsers() } catch (e: Exception) { emptyList() }
        }
        val tasks = async(SupervisorJob()) {
            try { repository.getTasks() } catch (e: Exception) { emptyList() }
        }

        DashboardData(users = users.await(), tasks = tasks.await())
    }
}
```

---

## runCatching パターン

```kotlin
class SafeRepository @Inject constructor(
    private val api: ApiService
) {
    suspend fun getUser(id: Int): Result<User> = runCatching {
        api.getUser(id)
    }.onFailure { e ->
        when (e) {
            is IOException -> Timber.w("Network error: ${e.message}")
            is HttpException -> Timber.w("HTTP ${e.code()}: ${e.message()}")
            else -> Timber.e(e, "Unexpected error")
        }
    }

    suspend fun updateUser(user: User): Result<Unit> = runCatching {
        api.updateUser(user.id, user)
    }.recover { e ->
        when (e) {
            is IOException -> throw RetryableException(e)
            else -> throw e
        }
    }
}

// Composeで使用
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val userResult by viewModel.user.collectAsStateWithLifecycle(null)

    userResult?.fold(
        onSuccess = { user -> UserCard(user) },
        onFailure = { error -> ErrorMessage(error.message ?: "エラー") }
    )
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `CoroutineExceptionHandler` | トップレベル例外捕捉 |
| `SupervisorJob` | 子の失敗を隔離 |
| `supervisorScope` | スコープ内で隔離 |
| `runCatching` | Result型で安全に |

- `CoroutineExceptionHandler`でグローバルエラーハンドリング
- `SupervisorJob`で並列処理の一部失敗を許容
- `runCatching`/`Result`で型安全なエラー処理
- `viewModelScope`は自動でSupervisorJob付き

---

8種類のAndroidアプリテンプレート（エラーハンドリング設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Channel](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-channel-2026)
- [Coroutine Testing](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-testing-2026)
- [エラーUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
