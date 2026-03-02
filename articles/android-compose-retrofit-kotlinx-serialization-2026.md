---
title: "Retrofit + kotlinx.serialization完全ガイド — Converter/sealed class/エラー処理"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit + kotlinx.serialization**（Converter設定、sealed classレスポンス、エラー処理）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
// implementation("com.squareup.retrofit2:retrofit:2.11.0")
// implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
// implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.0")

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        val json = Json {
            ignoreUnknownKeys = true
            coerceInputValues = true
            isLenient = true
        }
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()
    }
}
```

---

## データモデル

```kotlin
@Serializable
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    @SerialName("created_at") val createdAt: String,
    val role: UserRole = UserRole.USER
)

@Serializable
enum class UserRole {
    @SerialName("admin") ADMIN,
    @SerialName("user") USER,
    @SerialName("guest") GUEST
}

@Serializable
data class ApiResponse<T>(
    val data: T,
    val meta: Meta? = null
)

@Serializable
data class Meta(
    val page: Int,
    val totalPages: Int,
    val totalCount: Int
)
```

---

## APIインターフェース

```kotlin
interface UserApi {
    @GET("users")
    suspend fun getUsers(
        @Query("page") page: Int = 1,
        @Query("per_page") perPage: Int = 20
    ): ApiResponse<List<UserResponse>>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): ApiResponse<UserResponse>

    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): ApiResponse<UserResponse>
}

@Serializable
data class CreateUserRequest(
    val name: String,
    val email: String
)
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `asConverterFactory` | kotlinx.serialization変換 |
| `ignoreUnknownKeys` | 未知フィールド無視 |
| `@SerialName` | JSONキー名対応 |
| `coerceInputValues` | null→デフォルト値 |

- Gsonの代わりにkotlinx.serializationで型安全
- `ignoreUnknownKeys`でAPI変更に強い
- `@SerialName`でスネークケース対応
- `coerceInputValues`でnull安全なデシリアライズ

---

8種類のAndroidアプリテンプレート（API連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Retrofit + Flow](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-flow-2026)
- [Retrofit Multipart](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-multipart-2026)
