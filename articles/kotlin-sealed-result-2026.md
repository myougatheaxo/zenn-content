---
title: "Kotlin Result型パターン — sealed classでエラーハンドリング"
emoji: "🎯"
type: "tech"
topics: ["android", "kotlin", "errorhandling", "architecture"]
published: true
---

## この記事で学べること

Kotlinの**sealed class**を使ったResult型パターン（ApiResult、UiState、エラー分岐）を解説します。

---

## 基本のResult型

```kotlin
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>
    data class Error(val message: String, val code: Int? = null) : ApiResult<Nothing>
    data object Loading : ApiResult<Nothing>
}

// 使用
fun handleResult(result: ApiResult<User>) {
    when (result) {
        is ApiResult.Success -> showUser(result.data)
        is ApiResult.Error -> showError(result.message)
        ApiResult.Loading -> showLoading()
    }
}
```

---

## safeApiCall ラッパー

```kotlin
suspend fun <T> safeApiCall(
    call: suspend () -> T
): ApiResult<T> {
    return try {
        ApiResult.Success(call())
    } catch (e: HttpException) {
        ApiResult.Error(
            message = e.message(),
            code = e.code()
        )
    } catch (e: IOException) {
        ApiResult.Error(message = "ネットワークエラー")
    } catch (e: Exception) {
        ApiResult.Error(message = e.message ?: "不明なエラー")
    }
}

// Repository
class UserRepository(private val api: UserApi) {
    suspend fun getUser(id: String): ApiResult<User> = safeApiCall {
        api.getUser(id)
    }

    suspend fun updateUser(user: User): ApiResult<Unit> = safeApiCall {
        api.updateUser(user)
    }
}
```

---

## UiState パターン

```kotlin
sealed interface UiState<out T> {
    data object Idle : UiState<Nothing>
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState<User>>(UiState.Idle)
    val uiState: StateFlow<UiState<User>> = _uiState.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading

            _uiState.value = when (val result = repository.getUser(id)) {
                is ApiResult.Success -> UiState.Success(result.data)
                is ApiResult.Error -> UiState.Error(result.message)
                ApiResult.Loading -> UiState.Loading
            }
        }
    }
}
```

---

## Compose での分岐表示

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        UiState.Idle -> {
            // 初期状態
        }
        UiState.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is UiState.Success -> {
            UserContent(user = state.data)
        }
        is UiState.Error -> {
            ErrorContent(
                message = state.message,
                onRetry = { viewModel.loadUser("1") }
            )
        }
    }
}

@Composable
fun ErrorContent(message: String, onRetry: () -> Unit) {
    Column(
        Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(Icons.Default.Error, null, tint = MaterialTheme.colorScheme.error)
        Spacer(Modifier.height(8.dp))
        Text(message)
        Spacer(Modifier.height(16.dp))
        Button(onClick = onRetry) { Text("再試行") }
    }
}
```

---

## Result の変換・チェーン

```kotlin
// map: 成功値を変換
fun <T, R> ApiResult<T>.map(transform: (T) -> R): ApiResult<R> = when (this) {
    is ApiResult.Success -> ApiResult.Success(transform(data))
    is ApiResult.Error -> this
    ApiResult.Loading -> ApiResult.Loading
}

// flatMap: 成功値で次のAPIコール
suspend fun <T, R> ApiResult<T>.flatMap(
    transform: suspend (T) -> ApiResult<R>
): ApiResult<R> = when (this) {
    is ApiResult.Success -> transform(data)
    is ApiResult.Error -> this
    ApiResult.Loading -> ApiResult.Loading
}

// 使用例
suspend fun getUserWithPosts(userId: String): ApiResult<UserWithPosts> {
    return repository.getUser(userId).flatMap { user ->
        repository.getPosts(userId).map { posts ->
            UserWithPosts(user, posts)
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `sealed interface ApiResult` | API層のエラーハンドリング |
| `sealed interface UiState` | UI層の状態管理 |
| `safeApiCall` | try-catchの共通化 |
| `map`/`flatMap` | Result変換・チェーン |

- `when`の網羅性チェックで漏れを防止
- `Nothing`型で共通エラー型を実現
- ViewModelでApiResult→UiState変換
- Composeで`when`分岐して状態別UI表示

---

8種類のAndroidアプリテンプレート（Result型設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [エラーハンドリングUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
- [ViewModel + Hilt](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
- [コルーチン例外処理](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-exception-2026)
