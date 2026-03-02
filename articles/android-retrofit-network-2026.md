---
title: "Retrofit完全ガイド — ComposeアプリのAPI通信を実装する"
emoji: "🌐"
type: "tech"
topics: ["android", "kotlin", "retrofit", "network"]
published: true
---

## この記事で学べること

**Retrofit**を使ったAPI通信の実装方法（セットアップ・インターセプター・エラーハンドリング）を解説します。

---

## 依存関係

```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-kotlinx-serialization:2.11.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
}
```

---

## APIインターフェース

```kotlin
@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String
)

@Serializable
data class CreateUserRequest(
    val name: String,
    val email: String
)

interface UserApi {
    @GET("users")
    suspend fun getUsers(): List<User>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User

    @GET("users")
    suspend fun searchUsers(
        @Query("name") name: String,
        @Query("page") page: Int = 1
    ): List<User>

    @POST("users")
    suspend fun createUser(@Body request: CreateUserRequest): User

    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: Int, @Body request: CreateUserRequest): User

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: Int)
}
```

---

## Retrofitのセットアップ

```kotlin
object RetrofitClient {
    private val json = Json {
        ignoreUnknownKeys = true
        coerceInputValues = true
    }

    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                else HttpLoggingInterceptor.Level.NONE
    }

    private val okHttpClient = OkHttpClient.Builder()
        .addInterceptor(loggingInterceptor)
        .addInterceptor(AuthInterceptor())
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()

    val api: UserApi = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(okHttpClient)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
        .create(UserApi::class.java)
}
```

---

## 認証インターセプター

```kotlin
class AuthInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer ${TokenManager.getToken()}")
            .addHeader("Accept", "application/json")
            .build()
        return chain.proceed(request)
    }
}
```

---

## Repository

```kotlin
class UserRepository(private val api: UserApi) {

    suspend fun getUsers(): Result<List<User>> {
        return try {
            val users = api.getUsers()
            Result.success(users)
        } catch (e: HttpException) {
            Result.failure(Exception("HTTP ${e.code()}: ${e.message()}"))
        } catch (e: IOException) {
            Result.failure(Exception("ネットワークエラー"))
        }
    }

    suspend fun createUser(name: String, email: String): Result<User> {
        return try {
            val user = api.createUser(CreateUserRequest(name, email))
            Result.success(user)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

---

## ViewModel

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {

    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    init { loadUsers() }

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            repository.getUsers().fold(
                onSuccess = { _uiState.value = UserUiState.Success(it) },
                onFailure = { _uiState.value = UserUiState.Error(it.message ?: "不明なエラー") }
            )
        }
    }
}

sealed class UserUiState {
    data object Loading : UserUiState()
    data class Success(val users: List<User>) : UserUiState()
    data class Error(val message: String) : UserUiState()
}
```

---

## Compose UI

```kotlin
@Composable
fun UserListScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UserUiState.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is UserUiState.Success -> {
            LazyColumn {
                items(state.users) { user ->
                    ListItem(
                        headlineContent = { Text(user.name) },
                        supportingContent = { Text(user.email) }
                    )
                }
            }
        }
        is UserUiState.Error -> {
            Column(
                Modifier.fillMaxSize().padding(16.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text(state.message, color = MaterialTheme.colorScheme.error)
                Spacer(Modifier.height(16.dp))
                Button(onClick = { viewModel.loadUsers() }) {
                    Text("リトライ")
                }
            }
        }
    }
}
```

---

## まとめ

- `@GET`/`@POST`/`@PUT`/`@DELETE`でHTTPメソッド定義
- `@Path`/`@Query`/`@Body`でパラメータ指定
- `kotlinx-serialization`で型安全なJSON変換
- `Interceptor`で認証ヘッダー自動付与
- `Result`型でエラーハンドリング
- `sealed class UiState`でUI状態管理

---

8種類のAndroidアプリテンプレート（API通信設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutines & Flowガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
