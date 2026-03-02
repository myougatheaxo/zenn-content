---
title: "Result/Either パターン完全ガイド — エラーハンドリング設計"
emoji: "🎯"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "architecture"]
published: true
---

## この記事で学べること

**Result/Eitherパターン**（kotlin.Result、sealed classエラー型、Repository層設計、UI反映、テスト）を解説します。

---

## sealed classでエラー型定義

```kotlin
sealed interface AppResult<out T> {
    data class Success<T>(val data: T) : AppResult<T>
    data class Error(val error: AppError) : AppResult<Nothing>
}

sealed interface AppError {
    data class Network(val message: String) : AppError
    data class Server(val code: Int, val message: String) : AppError
    data class NotFound(val id: String) : AppError
    data object Unauthorized : AppError
    data class Unknown(val throwable: Throwable) : AppError
}
```

---

## safeApiCall

```kotlin
suspend fun <T> safeApiCall(block: suspend () -> T): AppResult<T> {
    return try {
        AppResult.Success(block())
    } catch (e: HttpException) {
        when (e.code()) {
            401 -> AppResult.Error(AppError.Unauthorized)
            404 -> AppResult.Error(AppError.NotFound(""))
            else -> AppResult.Error(AppError.Server(e.code(), e.message()))
        }
    } catch (e: IOException) {
        AppResult.Error(AppError.Network(e.message ?: "Network error"))
    } catch (e: Exception) {
        AppResult.Error(AppError.Unknown(e))
    }
}
```

---

## Repository

```kotlin
class UserRepository @Inject constructor(private val api: UserApi) {
    suspend fun getUser(id: String): AppResult<User> = safeApiCall {
        api.getUser(id)
    }

    suspend fun updateUser(user: User): AppResult<User> = safeApiCall {
        api.updateUser(user.id, user.toRequest())
    }
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState

    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            _uiState.value = when (val result = repository.getUser(id)) {
                is AppResult.Success -> UserUiState.Success(result.data)
                is AppResult.Error -> UserUiState.Error(result.error.toMessage())
            }
        }
    }
}

sealed interface UserUiState {
    data object Loading : UserUiState
    data class Success(val user: User) : UserUiState
    data class Error(val message: String) : UserUiState
}

fun AppError.toMessage(): String = when (this) {
    is AppError.Network -> "ネットワークに接続できません"
    is AppError.Server -> "サーバーエラー ($code)"
    is AppError.NotFound -> "データが見つかりません"
    AppError.Unauthorized -> "認証が必要です"
    is AppError.Unknown -> "予期しないエラーが発生しました"
}
```

---

## Compose画面

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        UserUiState.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is UserUiState.Success -> {
            UserContent(state.user)
        }
        is UserUiState.Error -> {
            ErrorContent(state.message, onRetry = { viewModel.loadUser("user-id") })
        }
    }
}

@Composable
fun ErrorContent(message: String, onRetry: () -> Unit) {
    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(Icons.Default.Warning, null, tint = MaterialTheme.colorScheme.error)
        Spacer(Modifier.height(8.dp))
        Text(message, style = MaterialTheme.typography.bodyLarge)
        Spacer(Modifier.height(16.dp))
        Button(onClick = onRetry) { Text("再試行") }
    }
}
```

---

## まとめ

| 層 | パターン |
|----|---------|
| API呼び出し | `safeApiCall` |
| エラー型 | `sealed interface AppError` |
| 結果型 | `AppResult<T>` |
| UI状態 | `sealed interface UiState` |

- sealed classで型安全なエラーハンドリング
- `safeApiCall`でAPI呼び出しを統一
- `AppError.toMessage()`でUI向けメッセージ変換
- `when`式でコンパイラが網羅性チェック

---

8種類のAndroidアプリテンプレート（エラーハンドリング実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [エラーUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [sealed class](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
