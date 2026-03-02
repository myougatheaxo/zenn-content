---
title: "Retrofit + OkHttp完全ガイド — API通信/インターセプター/認証"
emoji: "🌐"
type: "tech"
topics: ["android", "kotlin", "retrofit", "networking"]
published: true
---

## この記事で学べること

**Retrofit + OkHttp**（API定義、インターセプター、認証トークン管理、エラーハンドリング）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-kotlinx-serialization:2.11.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
}
```

---

## API定義

```kotlin
interface ApiService {

    @GET("users")
    suspend fun getUsers(): List<UserDto>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): UserDto

    @PUT("users/{id}")
    suspend fun updateUser(
        @Path("id") id: String,
        @Body user: UpdateUserRequest
    ): UserDto

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: String)

    @GET("search")
    suspend fun search(
        @Query("q") query: String,
        @Query("page") page: Int = 1,
        @Query("limit") limit: Int = 20
    ): SearchResponse

    @Multipart
    @POST("upload")
    suspend fun uploadImage(
        @Part image: MultipartBody.Part,
        @Part("description") description: RequestBody
    ): UploadResponse
}

@Serializable
data class UserDto(
    val id: String,
    val name: String,
    val email: String
)
```

---

## OkHttpClient設定

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor
    ): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(authInterceptor)
        .addInterceptor(
            HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG)
                    HttpLoggingInterceptor.Level.BODY
                else HttpLoggingInterceptor.Level.NONE
            }
        )
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/v1/")
            .client(okHttpClient)
            .addConverterFactory(
                Json { ignoreUnknownKeys = true }
                    .asConverterFactory("application/json".toMediaType())
            )
            .build()

    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService =
        retrofit.create(ApiService::class.java)
}
```

---

## 認証インターセプター

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenProvider: TokenProvider
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getAccessToken()

        val request = chain.request().newBuilder().apply {
            token?.let { addHeader("Authorization", "Bearer $it") }
            addHeader("Accept", "application/json")
        }.build()

        val response = chain.proceed(request)

        // 401なら トークンリフレッシュ
        if (response.code == 401) {
            response.close()
            val newToken = runBlocking { tokenProvider.refreshToken() }
            if (newToken != null) {
                val newRequest = request.newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
                return chain.proceed(newRequest)
            }
        }

        return response
    }
}
```

---

## Repository + エラーハンドリング

```kotlin
class UserRepository @Inject constructor(
    private val api: ApiService
) {
    suspend fun getUsers(): Result<List<User>> = safeApiCall {
        api.getUsers().map { it.toDomain() }
    }

    suspend fun createUser(name: String, email: String): Result<User> = safeApiCall {
        api.createUser(CreateUserRequest(name, email)).toDomain()
    }
}

suspend fun <T> safeApiCall(call: suspend () -> T): Result<T> =
    try {
        Result.success(call())
    } catch (e: HttpException) {
        val message = when (e.code()) {
            400 -> "リクエストエラー"
            401 -> "認証エラー"
            403 -> "アクセス拒否"
            404 -> "見つかりません"
            500 -> "サーバーエラー"
            else -> "通信エラー (${e.code()})"
        }
        Result.failure(ApiException(e.code(), message))
    } catch (e: IOException) {
        Result.failure(ApiException(0, "ネットワーク接続を確認してください"))
    }

class ApiException(val code: Int, override val message: String) : Exception(message)
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `@GET/@POST` | HTTP メソッド定義 |
| `Interceptor` | リクエスト/レスポンス加工 |
| `Authenticator` | トークンリフレッシュ |
| `LoggingInterceptor` | デバッグログ |
| `safeApiCall` | エラーハンドリング |

- `kotlinx-serialization`でJSON変換
- `Interceptor`で認証ヘッダー自動付与
- `safeApiCall`でHTTP/ネットワークエラー統一処理
- BuildConfig.DEBUGでログレベル切替

---

8種類のAndroidアプリテンプレート（API通信対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Ktor Client](https://zenn.dev/myougatheaxo/articles/android-compose-ktor-multiplatform-2026)
- [sealed Result](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-result-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
