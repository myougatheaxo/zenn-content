---
title: "Retrofit CallAdapter完全ガイド — Result型/Either/カスタムアダプター"
emoji: "🔌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit CallAdapter**（Result型ラッパー、カスタムCallAdapter、Either型パターン）を解説します。

---

## Result型 CallAdapter

```kotlin
// Result<T>を返すCallAdapter
class ResultCallAdapterFactory : CallAdapter.Factory() {
    override fun get(
        returnType: Type, annotations: Array<out Annotation>, retrofit: Retrofit
    ): CallAdapter<*, *>? {
        if (getRawType(returnType) != Call::class.java) return null
        val responseType = getParameterUpperBound(0, returnType as ParameterizedType)
        if (getRawType(responseType) != Result::class.java) return null
        val successType = getParameterUpperBound(0, responseType as ParameterizedType)
        return ResultCallAdapter<Any>(successType)
    }
}

class ResultCallAdapter<T>(private val responseType: Type) : CallAdapter<T, Call<Result<T>>> {
    override fun responseType() = responseType
    override fun adapt(call: Call<T>): Call<Result<T>> = ResultCall(call)
}

class ResultCall<T>(private val delegate: Call<T>) : Call<Result<T>> {
    override fun enqueue(callback: Callback<Result<T>>) {
        delegate.enqueue(object : Callback<T> {
            override fun onResponse(call: Call<T>, response: Response<T>) {
                val result = if (response.isSuccessful) {
                    Result.success(response.body()!!)
                } else {
                    Result.failure(HttpException(response))
                }
                callback.onResponse(this@ResultCall, Response.success(result))
            }
            override fun onFailure(call: Call<T>, t: Throwable) {
                callback.onResponse(
                    this@ResultCall, Response.success(Result.failure(t))
                )
            }
        })
    }
    override fun clone() = ResultCall(delegate.clone())
    override fun execute(): Response<Result<T>> = throw UnsupportedOperationException()
    override fun isExecuted() = delegate.isExecuted
    override fun cancel() = delegate.cancel()
    override fun isCanceled() = delegate.isCanceled
    override fun request() = delegate.request()
    override fun timeout() = delegate.timeout()
}
```

---

## APIインターフェース

```kotlin
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): Result<UserResponse>

    @GET("users")
    suspend fun getUsers(): Result<List<UserResponse>>
}

// Retrofit設定
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addCallAdapterFactory(ResultCallAdapterFactory())
            .addConverterFactory(GsonConverterFactory.create())
            .build()
}
```

---

## ViewModel使用

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val api: UserApi
) : ViewModel() {
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState = _uiState.asStateFlow()

    fun loadUser(id: Long) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            api.getUser(id)
                .onSuccess { _uiState.value = UserUiState.Success(it) }
                .onFailure { _uiState.value = UserUiState.Error(it.message ?: "Unknown") }
        }
    }
}

sealed interface UserUiState {
    data object Loading : UserUiState
    data class Success(val user: UserResponse) : UserUiState
    data class Error(val message: String) : UserUiState
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `CallAdapter.Factory` | アダプター生成 |
| `CallAdapter` | Call変換定義 |
| `ResultCall` | Result型ラップ |
| `Result<T>` | 成功/失敗を型安全に |

- `CallAdapter`でRetrofitのレスポンスをResult型に変換
- try-catchなしでエラーハンドリング
- `onSuccess`/`onFailure`でシンプルな分岐
- ViewModelでUiState変換に活用

---

8種類のAndroidアプリテンプレート（API連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Retrofit + Flow](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-flow-2026)
- [Retrofit kotlinx.serialization](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-kotlinx-serialization-2026)
