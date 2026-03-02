---
title: "Ktor Client完全ガイド — Android HTTP通信の新定番"
emoji: "🌐"
type: "tech"
topics: ["android", "kotlin", "ktor", "network"]
published: true
---

## この記事で学べること

**Ktor Client**を使ったAndroid HTTP通信（GET/POST、認証、シリアライズ）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("io.ktor:ktor-client-android:3.0.3")
    implementation("io.ktor:ktor-client-content-negotiation:3.0.3")
    implementation("io.ktor:ktor-serialization-kotlinx-json:3.0.3")
    implementation("io.ktor:ktor-client-logging:3.0.3")
    implementation("io.ktor:ktor-client-auth:3.0.3")
}
```

---

## クライアント設定

```kotlin
val client = HttpClient(Android) {
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            prettyPrint = true
        })
    }
    install(Logging) {
        level = LogLevel.BODY
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 15000
        connectTimeoutMillis = 10000
    }
    defaultRequest {
        url("https://api.example.com/")
        contentType(ContentType.Application.Json)
    }
}
```

---

## GET/POST リクエスト

```kotlin
@Serializable
data class User(val id: Long, val name: String, val email: String)

@Serializable
data class CreateUserRequest(val name: String, val email: String)

class UserApi(private val client: HttpClient) {

    suspend fun getUsers(): List<User> {
        return client.get("users").body()
    }

    suspend fun getUser(id: Long): User {
        return client.get("users/$id").body()
    }

    suspend fun createUser(request: CreateUserRequest): User {
        return client.post("users") {
            setBody(request)
        }.body()
    }

    suspend fun deleteUser(id: Long) {
        client.delete("users/$id")
    }
}
```

---

## 認証（Bearer Token）

```kotlin
val client = HttpClient(Android) {
    install(Auth) {
        bearer {
            loadTokens {
                BearerTokens(
                    accessToken = tokenStorage.getAccessToken(),
                    refreshToken = tokenStorage.getRefreshToken()
                )
            }
            refreshTokens {
                val response = client.post("auth/refresh") {
                    setBody(mapOf("refresh_token" to oldTokens?.refreshToken))
                }.body<TokenResponse>()
                tokenStorage.saveTokens(response)
                BearerTokens(response.accessToken, response.refreshToken)
            }
        }
    }
}
```

---

## Repository + ViewModel

```kotlin
class UserRepository(private val api: UserApi) {
    suspend fun getUsers(): Result<List<User>> {
        return try {
            Result.success(api.getUsers())
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<User>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<User>>> = _uiState.asStateFlow()

    init { loadUsers() }

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            repository.getUsers()
                .onSuccess { _uiState.value = UiState.Success(it) }
                .onFailure { _uiState.value = UiState.Error(it.message ?: "エラー") }
        }
    }
}
```

---

## まとめ

- Ktor ClientはKotlin純正のHTTPクライアント
- `ContentNegotiation`でkotlinx-serialization自動変換
- `install(Auth) { bearer {} }`でトークン自動リフレッシュ
- `HttpTimeout`でタイムアウト設定
- suspend関数でシンプルな非同期API
- KMM（Kotlin Multiplatform）でも同じコードが使える

---

8種類のAndroidアプリテンプレート（ネットワーク通信設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit通信ガイド](https://zenn.dev/myougatheaxo/articles/android-retrofit-network-2026)
- [Coroutines/Flowガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
- [例外処理ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-exception-2026)
