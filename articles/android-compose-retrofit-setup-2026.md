---
title: "Retrofit + Compose連携ガイド — API通信の定番パターン"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit**でのAPI通信とCompose連携（セットアップ、エラー処理、インターセプター）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
}
```

---

## APIインターフェース

```kotlin
interface UserApi {
    @GET("users")
    suspend fun getUsers(): List<UserDto>

    @GET("users/{id}")
    suspend fun getUserById(@Path("id") id: String): UserDto

    @POST("users")
    suspend fun createUser(@Body request: CreateUserRequest): UserDto

    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: String, @Body request: UpdateUserRequest): UserDto

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: String)

    @GET("users")
    suspend fun searchUsers(
        @Query("q") query: String,
        @Query("page") page: Int = 1,
        @Query("limit") limit: Int = 20
    ): PaginatedResponse<UserDto>
}
```

---

## Retrofit構築

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .addInterceptor { chain ->
                val request = chain.request().newBuilder()
                    .addHeader("Authorization", "Bearer ${TokenManager.getToken()}")
                    .addHeader("Accept", "application/json")
                    .build()
                chain.proceed(request)
            }
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/v1/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi {
        return retrofit.create(UserApi::class.java)
    }
}
```

---

## Repository + エラーハンドリング

```kotlin
class UserRepository(private val api: UserApi) {

    suspend fun getUsers(): Result<List<User>> = safeApiCall {
        api.getUsers().map { it.toDomain() }
    }

    suspend fun createUser(name: String, email: String): Result<User> = safeApiCall {
        api.createUser(CreateUserRequest(name, email)).toDomain()
    }
}

suspend fun <T> safeApiCall(call: suspend () -> T): Result<T> {
    return try {
        Result.success(call())
    } catch (e: HttpException) {
        val message = when (e.code()) {
            401 -> "認証エラー"
            403 -> "アクセス権限がありません"
            404 -> "データが見つかりません"
            500 -> "サーバーエラー"
            else -> "通信エラー (${e.code()})"
        }
        Result.failure(Exception(message))
    } catch (e: IOException) {
        Result.failure(Exception("ネットワーク接続を確認してください"))
    }
}
```

---

## まとめ

- `suspend fun`で非同期API呼び出し
- `@GET`/`@POST`/`@PUT`/`@DELETE`でHTTPメソッド指定
- `@Path`/`@Query`/`@Body`でパラメータ設定
- OkHttp Interceptorで認証ヘッダー自動付与
- `HttpLoggingInterceptor`でデバッグログ
- `safeApiCall`パターンでエラーを統一的に処理

---

8種類のAndroidアプリテンプレート（API通信実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Ktor Clientガイド](https://zenn.dev/myougatheaxo/articles/android-compose-ktor-client-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
- [Hilt依存性注入](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
