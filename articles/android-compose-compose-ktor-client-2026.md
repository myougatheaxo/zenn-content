---
title: "Compose Ktor Client完全ガイド — HTTPリクエスト/シリアライズ/WebSocket/認証"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ktor"]
published: true
---

## この記事で学べること

**Compose Ktor Client**（HTTPリクエスト、kotlinx.serialization、WebSocket、認証プラグイン）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation("io.ktor:ktor-client-android:2.3.12")
    implementation("io.ktor:ktor-client-content-negotiation:2.3.12")
    implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.12")
    implementation("io.ktor:ktor-client-logging:2.3.12")
    implementation("io.ktor:ktor-client-auth:2.3.12")
}
```

---

## HTTPクライアント

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideKtorClient(): HttpClient = HttpClient(Android) {
        install(ContentNegotiation) {
            json(Json {
                prettyPrint = true
                isLenient = true
                ignoreUnknownKeys = true
            })
        }
        install(Logging) {
            level = LogLevel.BODY
        }
        install(HttpTimeout) {
            requestTimeoutMillis = 15_000
            connectTimeoutMillis = 10_000
        }
        defaultRequest {
            url("https://api.example.com/v1/")
            contentType(ContentType.Application.Json)
        }
    }
}

// Repository
class UserRepository @Inject constructor(private val client: HttpClient) {
    suspend fun getUsers(): List<UserDto> =
        client.get("users").body()

    suspend fun getUser(id: String): UserDto =
        client.get("users/$id").body()

    suspend fun createUser(user: CreateUserRequest): UserDto =
        client.post("users") { setBody(user) }.body()

    suspend fun deleteUser(id: String) {
        client.delete("users/$id")
    }
}

@Serializable
data class UserDto(val id: String, val name: String, val email: String)

@Serializable
data class CreateUserRequest(val name: String, val email: String)
```

---

## 認証プラグイン

```kotlin
fun provideAuthClient(tokenStorage: TokenStorage): HttpClient = HttpClient(Android) {
    install(ContentNegotiation) { json() }
    install(Auth) {
        bearer {
            loadTokens {
                BearerTokens(tokenStorage.accessToken, tokenStorage.refreshToken)
            }
            refreshTokens {
                val response: TokenResponse = client.post("auth/refresh") {
                    setBody(RefreshRequest(tokenStorage.refreshToken))
                }.body()
                tokenStorage.save(response.accessToken, response.refreshToken)
                BearerTokens(response.accessToken, response.refreshToken)
            }
        }
    }
}
```

---

## Compose連携

```kotlin
@HiltViewModel
class UsersViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    var uiState by mutableStateOf<UiState<List<UserDto>>>(UiState.Loading)
        private set

    init { loadUsers() }

    private fun loadUsers() {
        viewModelScope.launch {
            uiState = try {
                UiState.Success(repository.getUsers())
            } catch (e: Exception) {
                UiState.Error(e.message ?: "エラー")
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `HttpClient` | HTTPクライアント |
| `ContentNegotiation` | JSON変換 |
| `Auth` | 認証プラグイン |
| `HttpTimeout` | タイムアウト設定 |

- Ktor ClientはKotlinネイティブのHTTPクライアント
- `ContentNegotiation`でkotlinx.serializationと連携
- `Auth`プラグインでBearer tokenの自動リフレッシュ
- KMP対応でiOS/Desktopでも同じコードが使える

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Retrofit](https://zenn.dev/myougatheaxo/articles/android-compose-compose-retrofit-2026)
- [Compose WebSocket](https://zenn.dev/myougatheaxo/articles/android-compose-compose-websocket-2026)
- [Compose GraphQL](https://zenn.dev/myougatheaxo/articles/android-compose-compose-graphql-2026)
