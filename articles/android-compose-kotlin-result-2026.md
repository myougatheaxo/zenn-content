---
title: "Kotlin Result完全ガイド — Result型/runCatching/map/recover/エラーハンドリング"
emoji: "✅"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "errorhandling"]
published: true
---

## この記事で学べること

**Kotlin Result**（Result型、runCatching、map/recover、fold、カスタムResult）を解説します。

---

## 基本的なResult

```kotlin
fun parseNumber(input: String): Result<Int> = runCatching {
    input.toInt()
}

fun main() {
    val result = parseNumber("42")
    result.onSuccess { println("成功: $it") }
        .onFailure { println("失敗: ${it.message}") }

    // 値の取得
    val value = result.getOrNull()     // Int? (失敗時null)
    val safe = result.getOrDefault(0)  // Int (失敗時デフォルト)
    val mapped = result.getOrElse { -1 } // Int (失敗時ラムダ)
}
```

---

## チェーン操作

```kotlin
class UserService @Inject constructor(
    private val api: UserApi,
    private val db: UserDao
) {
    suspend fun getUser(id: String): Result<UserDto> = runCatching {
        api.fetchUser(id)
    }.recoverCatching {
        // API失敗時はローカルDBから取得
        db.getUser(id) ?: throw Exception("User not found")
    }.map { entity ->
        UserDto(id = entity.id, name = entity.name)
    }

    suspend fun updateProfile(name: String): Result<Unit> = runCatching {
        require(name.isNotBlank()) { "名前は必須です" }
        api.updateProfile(name)
    }
}

// ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userService: UserService
) : ViewModel() {
    var uiState by mutableStateOf<UiState>(UiState.Loading)
        private set

    fun loadUser(id: String) {
        viewModelScope.launch {
            uiState = userService.getUser(id).fold(
                onSuccess = { UiState.Success(it) },
                onFailure = { UiState.Error(it.message ?: "エラー") }
            )
        }
    }
}
```

---

## カスタムResult

```kotlin
sealed class AppResult<out T> {
    data class Success<T>(val data: T) : AppResult<T>()
    data class Error(val code: ErrorCode, val message: String) : AppResult<Nothing>()

    fun <R> map(transform: (T) -> R): AppResult<R> = when (this) {
        is Success -> Success(transform(data))
        is Error -> this
    }

    fun onSuccess(action: (T) -> Unit): AppResult<T> {
        if (this is Success) action(data)
        return this
    }

    fun onError(action: (ErrorCode, String) -> Unit): AppResult<T> {
        if (this is Error) action(code, message)
        return this
    }
}

enum class ErrorCode { NETWORK, AUTH, NOT_FOUND, UNKNOWN }

// 使用
suspend fun login(email: String, password: String): AppResult<User> {
    return try {
        val user = api.login(email, password)
        AppResult.Success(user)
    } catch (e: HttpException) {
        when (e.code()) {
            401 -> AppResult.Error(ErrorCode.AUTH, "認証失敗")
            404 -> AppResult.Error(ErrorCode.NOT_FOUND, "ユーザーが見つかりません")
            else -> AppResult.Error(ErrorCode.UNKNOWN, e.message())
        }
    } catch (e: IOException) {
        AppResult.Error(ErrorCode.NETWORK, "ネットワークエラー")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `runCatching` | Result生成 |
| `map` | 値変換 |
| `recover` | エラー回復 |
| `fold` | 成功/失敗分岐 |

- `runCatching`で例外をResultに変換
- `map`/`mapCatching`でチェーン変換
- `recover`/`recoverCatching`でフォールバック
- `fold`で成功/失敗を統一的にハンドリング

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin SealedClass](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-sealed-class-2026)
- [Coroutine Cancellation](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-cancellation-2026)
- [Compose CleanArchitecture](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clean-architecture-2026)
