---
title: "Coroutine ExceptionHandler完全ガイド — 例外処理/SupervisorJob/エラーハンドリング"
emoji: "⚠️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Coroutine ExceptionHandler**（CoroutineExceptionHandler、SupervisorJob、try-catch戦略、グローバルエラー処理）を解説します。

---

## 基本ExceptionHandler

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val repository: DataRepository
) : ViewModel() {

    private val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        _error.value = throwable.message
    }

    private val _error = MutableStateFlow<String?>(null)
    val error: StateFlow<String?> = _error.asStateFlow()

    fun loadData() {
        viewModelScope.launch(exceptionHandler) {
            val data = repository.fetchData() // 例外が発生してもクラッシュしない
            _data.value = data
        }
    }

    fun clearError() { _error.value = null }
}
```

---

## SupervisorJob

```kotlin
@HiltViewModel
class ParallelViewModel @Inject constructor() : ViewModel() {

    fun loadParallel() {
        // SupervisorJobで子コルーチンの失敗が他に影響しない
        viewModelScope.launch {
            supervisorScope {
                val job1 = launch {
                    try { fetchUserProfile() }
                    catch (e: Exception) { /* job1の失敗はjob2に影響しない */ }
                }
                val job2 = launch {
                    try { fetchNotifications() }
                    catch (e: Exception) { /* job2の失敗はjob1に影響しない */ }
                }
            }
        }
    }
}
```

---

## Result型パターン

```kotlin
sealed class AppResult<out T> {
    data class Success<T>(val data: T) : AppResult<T>()
    data class Error(val message: String, val exception: Throwable? = null) : AppResult<Nothing>()
    data object Loading : AppResult<Nothing>()
}

suspend fun <T> safeApiCall(call: suspend () -> T): AppResult<T> =
    try {
        AppResult.Success(call())
    } catch (e: HttpException) {
        AppResult.Error("サーバーエラー: ${e.code()}", e)
    } catch (e: IOException) {
        AppResult.Error("ネットワークエラー", e)
    } catch (e: Exception) {
        AppResult.Error("不明なエラー", e)
    }

// 使用
viewModelScope.launch {
    _state.value = AppResult.Loading
    _state.value = safeApiCall { repository.fetchItems() }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `CoroutineExceptionHandler` | グローバル例外処理 |
| `SupervisorJob` | 子の失敗を隔離 |
| `supervisorScope` | スコープ単位の隔離 |
| `Result`型 | 型安全なエラー処理 |

- `CoroutineExceptionHandler`は`launch`のみ有効（`async`は`await`で取得）
- `SupervisorJob`で並行タスクの失敗を隔離
- `Result`型パターンで型安全なエラーハンドリング
- `viewModelScope`は内部で`SupervisorJob`を使用

---

8種類のAndroidアプリテンプレート（エラーハンドリング対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Mutex](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-mutex-2026)
- [Flow Retry](https://zenn.dev/myougatheaxo/articles/android-compose-flow-retry-2026)
- [Coroutine Supervisor](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-supervisor-2026)
