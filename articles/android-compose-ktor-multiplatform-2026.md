---
title: "Ktor Client完全ガイド — HTTP通信/認証/WebSocket"
emoji: "🌍"
type: "tech"
topics: ["android", "kotlin", "ktor", "network"]
published: true
---

## この記事で学べること

**Ktor Client**のHTTP通信（リクエスト/レスポンス、認証、WebSocket、マルチプラットフォーム対応）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.ktor:ktor-client-core:2.3.12")
    implementation("io.ktor:ktor-client-okhttp:2.3.12") // Android
    implementation("io.ktor:ktor-client-content-negotiation:2.3.12")
    implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.12")
    implementation("io.ktor:ktor-client-logging:2.3.12")
    implementation("io.ktor:ktor-client-auth:2.3.12")
}
```

---

## HttpClient設定

```kotlin
val client = HttpClient(OkHttp) {
    // JSON シリアライゼーション
    install(ContentNegotiation) {
        json(Json {
            prettyPrint = true
            isLenient = true
            ignoreUnknownKeys = true
        })
    }

    // ログ
    install(Logging) {
        logger = Logger.ANDROID
        level = LogLevel.BODY
    }

    // デフォルトリクエスト設定
    defaultRequest {
        url("https://api.example.com/v1/")
        header("Accept", "application/json")
    }

    // タイムアウト
    install(HttpTimeout) {
        requestTimeoutMillis = 30_000
        connectTimeoutMillis = 10_000
        socketTimeoutMillis = 30_000
    }
}
```

---

## CRUD操作

```kotlin
@Serializable
data class User(val id: String, val name: String, val email: String)

class UserApiClient(private val client: HttpClient) {

    // GET
    suspend fun getUsers(): List<User> {
        return client.get("users").body()
    }

    suspend fun getUser(id: String): User {
        return client.get("users/$id").body()
    }

    // POST
    suspend fun createUser(name: String, email: String): User {
        return client.post("users") {
            contentType(ContentType.Application.Json)
            setBody(mapOf("name" to name, "email" to email))
        }.body()
    }

    // PUT
    suspend fun updateUser(id: String, name: String): User {
        return client.put("users/$id") {
            contentType(ContentType.Application.Json)
            setBody(mapOf("name" to name))
        }.body()
    }

    // DELETE
    suspend fun deleteUser(id: String) {
        client.delete("users/$id")
    }

    // クエリパラメータ
    suspend fun searchUsers(query: String, page: Int = 1): List<User> {
        return client.get("users/search") {
            parameter("q", query)
            parameter("page", page)
        }.body()
    }
}
```

---

## Bearer認証

```kotlin
val client = HttpClient(OkHttp) {
    install(Auth) {
        bearer {
            loadTokens {
                BearerTokens(
                    accessToken = tokenStore.getAccessToken(),
                    refreshToken = tokenStore.getRefreshToken()
                )
            }

            refreshTokens {
                val response = client.post("auth/refresh") {
                    setBody(mapOf("refresh_token" to oldTokens?.refreshToken))
                }.body<TokenResponse>()

                tokenStore.saveTokens(response.accessToken, response.refreshToken)
                BearerTokens(response.accessToken, response.refreshToken)
            }
        }
    }
}
```

---

## WebSocket

```kotlin
suspend fun connectWebSocket() {
    client.webSocket("wss://api.example.com/ws") {
        // メッセージ送信
        send(Frame.Text(Json.encodeToString(WsMessage("subscribe", "chat"))))

        // メッセージ受信
        for (frame in incoming) {
            when (frame) {
                is Frame.Text -> {
                    val text = frame.readText()
                    val message = Json.decodeFromString<WsMessage>(text)
                    handleMessage(message)
                }
                is Frame.Close -> break
                else -> {}
            }
        }
    }
}

// Flow変換
fun observeMessages(): Flow<WsMessage> = callbackFlow {
    client.webSocket("wss://api.example.com/ws") {
        for (frame in incoming) {
            if (frame is Frame.Text) {
                val msg = Json.decodeFromString<WsMessage>(frame.readText())
                trySend(msg)
            }
        }
    }
    awaitClose()
}
```

---

## エラーハンドリング

```kotlin
suspend fun <T> safeApiCall(call: suspend () -> T): ApiResult<T> {
    return try {
        ApiResult.Success(call())
    } catch (e: ClientRequestException) { // 4xx
        val errorBody = e.response.bodyAsText()
        ApiResult.Error("Client error: ${e.response.status}", errorBody)
    } catch (e: ServerResponseException) { // 5xx
        ApiResult.Error("Server error: ${e.response.status}")
    } catch (e: HttpRequestTimeoutException) {
        ApiResult.Error("Request timeout")
    } catch (e: ConnectTimeoutException) {
        ApiResult.Error("Connection timeout")
    } catch (e: Exception) {
        ApiResult.Error(e.message ?: "Unknown error")
    }
}

// 使用
val result = safeApiCall { api.getUsers() }
```

---

## まとめ

| 機能 | プラグイン |
|------|----------|
| JSON変換 | `ContentNegotiation` |
| ログ | `Logging` |
| 認証 | `Auth` (Bearer/Basic) |
| タイムアウト | `HttpTimeout` |
| WebSocket | 組み込みサポート |

- `kotlinx.serialization`でJSON自動変換
- Bearer認証のトークンリフレッシュ自動化
- WebSocketをFlow変換でCompose連携
- KMPで同じコードをiOS/Desktop/Webで共有可能

---

8種類のAndroidアプリテンプレート（ネットワーク設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit設定](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-setup-2026)
- [KMP共有UI](https://zenn.dev/myougatheaxo/articles/android-compose-kmp-shared-ui-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
