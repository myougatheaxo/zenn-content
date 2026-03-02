---
title: "Retrofit + Flow完全ガイド — Flow返却/リトライ/ポーリング/エラーハンドリング"
emoji: "🌊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit + Flow**（Flow返却、リトライ、ポーリング、統一エラーハンドリング）を解説します。

---

## API→Flow変換

```kotlin
interface ApiService {
    @GET("users")
    suspend fun getUsers(): List<User>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): User
}

class UserRepository @Inject constructor(private val api: ApiService) {
    fun getUsers(): Flow<Result<List<User>>> = flow {
        emit(Result.Loading)
        try {
            val users = api.getUsers()
            emit(Result.Success(users))
        } catch (e: Exception) {
            emit(Result.Error(e))
        }
    }

    // リトライ付き
    fun getUsersWithRetry(): Flow<Result<List<User>>> = flow {
        emit(Result.Loading)
        val users = api.getUsers()
        emit(Result.Success(users))
    }.retryWhen { cause, attempt ->
        if (attempt < 3 && cause is IOException) {
            delay(1000 * (attempt + 1))
            true
        } else false
    }.catch { emit(Result.Error(it)) }
}

sealed class Result<out T> {
    data object Loading : Result<Nothing>()
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
}
```

---

## ポーリング

```kotlin
fun pollPrices(interval: Long = 5000L): Flow<List<Price>> = flow {
    while (true) {
        val prices = api.getPrices()
        emit(prices)
        delay(interval)
    }
}.catch { emit(emptyList()) }
```

---

## ViewModel

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    private val refreshTrigger = MutableSharedFlow<Unit>(replay = 1).apply { tryEmit(Unit) }

    val users = refreshTrigger
        .flatMapLatest { repository.getUsers() }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), Result.Loading)

    fun refresh() { refreshTrigger.tryEmit(Unit) }
}

@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val result by viewModel.users.collectAsStateWithLifecycle()

    when (val state = result) {
        is Result.Loading -> CircularProgressIndicator()
        is Result.Success -> {
            LazyColumn {
                items(state.data) { user ->
                    ListItem(headlineContent = { Text(user.name) })
                }
            }
        }
        is Result.Error -> Text("エラー: ${state.exception.message}")
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `flow { }` | API→Flow変換 |
| `retryWhen` | リトライ制御 |
| `catch` | エラーハンドリング |
| `flatMapLatest` | リフレッシュ |

- `flow { }`でsuspend APIをFlowに変換
- `retryWhen`で回数/条件付きリトライ
- `catch`でエラーをResultに変換
- `flatMapLatest`でリフレッシュトリガー

---

8種類のAndroidアプリテンプレート（API連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Coroutine Exception](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
