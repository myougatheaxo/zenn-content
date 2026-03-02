---
title: "Coroutine例外処理ガイド — supervisorScope/CoroutineExceptionHandler"
emoji: "⚠️"
type: "tech"
topics: ["android", "kotlin", "coroutine", "errorhandling"]
published: true
---

## この記事で学べること

Kotlinコルーチンの**例外処理パターン**（try-catch、supervisorScope、CoroutineExceptionHandler）を解説します。

---

## 基本のtry-catch

```kotlin
class UserViewModel : ViewModel() {

    fun loadUser(id: String) {
        viewModelScope.launch {
            try {
                val user = repository.getUser(id)
                _uiState.value = UiState.Success(user)
            } catch (e: HttpException) {
                _uiState.value = UiState.Error("サーバーエラー: ${e.code()}")
            } catch (e: IOException) {
                _uiState.value = UiState.Error("ネットワークエラー")
            } catch (e: Exception) {
                _uiState.value = UiState.Error("予期せぬエラー")
            }
        }
    }
}
```

---

## supervisorScope

```kotlin
// supervisorScope: 子コルーチンの失敗が他の子に影響しない
fun loadDashboard() {
    viewModelScope.launch {
        supervisorScope {
            val userDeferred = async { repository.getUser() }
            val newsDeferred = async { repository.getNews() }
            val statsDeferred = async { repository.getStats() }

            val user = try { userDeferred.await() } catch (e: Exception) { null }
            val news = try { newsDeferred.await() } catch (e: Exception) { emptyList() }
            val stats = try { statsDeferred.await() } catch (e: Exception) { null }

            _uiState.value = DashboardState(user, news, stats)
        }
    }
}
```

---

## CoroutineExceptionHandler

```kotlin
class AppViewModel : ViewModel() {

    private val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        _error.value = when (throwable) {
            is HttpException -> "サーバーエラー"
            is IOException -> "ネットワークエラー"
            else -> "予期せぬエラー: ${throwable.message}"
        }
    }

    fun doWork() {
        viewModelScope.launch(exceptionHandler) {
            // 例外はexceptionHandlerでキャッチされる
            repository.saveData()
        }
    }
}
```

---

## Result型パターン

```kotlin
// Repositoryでラップ
class UserRepository(private val api: UserApi) {

    suspend fun getUser(id: String): Result<User> {
        return try {
            val user = api.getUser(id)
            Result.success(user)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// ViewModelで使用
fun loadUser(id: String) {
    viewModelScope.launch {
        repository.getUser(id)
            .onSuccess { user ->
                _uiState.value = UiState.Success(user)
            }
            .onFailure { e ->
                _uiState.value = UiState.Error(e.message ?: "エラー")
            }
    }
}
```

---

## runCatchingパターン

```kotlin
fun processData() {
    viewModelScope.launch {
        runCatching { repository.fetchData() }
            .mapCatching { data -> transform(data) }
            .mapCatching { transformed -> repository.save(transformed) }
            .onSuccess { _uiState.value = UiState.Success(it) }
            .onFailure { _uiState.value = UiState.Error(it.message ?: "エラー") }
    }
}
```

---

## キャンセルの伝播

```kotlin
// CancellationExceptionは再スローすること
suspend fun safeFetch(): Data {
    return try {
        api.fetchData()
    } catch (e: CancellationException) {
        throw e // キャンセルは再スロー必須
    } catch (e: Exception) {
        fallbackData()
    }
}

// withTimeoutはCancellationExceptionをスロー
fun loadWithTimeout() {
    viewModelScope.launch {
        try {
            val result = withTimeout(5000) {
                repository.fetchData()
            }
            _uiState.value = UiState.Success(result)
        } catch (e: TimeoutCancellationException) {
            _uiState.value = UiState.Error("タイムアウト")
        }
    }
}
```

---

## まとめ

- `try-catch`が基本の例外処理
- `supervisorScope`で子コルーチンを独立
- `CoroutineExceptionHandler`でグローバルハンドリング
- `Result<T>`/`runCatching`でチェーン処理
- `CancellationException`は必ず再スロー
- `withTimeout`でタイムアウト処理

---

8種類のAndroidアプリテンプレート（例外処理設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutines/Flow完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
- [エラーハンドリングパターン](https://zenn.dev/myougatheaxo/articles/android-error-handling-patterns-2026)
- [Retrofit通信ガイド](https://zenn.dev/myougatheaxo/articles/android-retrofit-network-2026)
