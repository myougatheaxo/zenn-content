---
title: "Retrofit Interceptor完全ガイド — 認証/ログ/リトライ/キャッシュ"
emoji: "🔧"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "network"]
published: true
---

## この記事で学べること

**Retrofit Interceptor**（認証ヘッダー、ログ出力、リトライ、キャッシュ制御）を解説します。

---

## 認証Interceptor

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenManager: TokenManager
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenManager.getAccessToken()
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .addHeader("Accept", "application/json")
            .build()

        val response = chain.proceed(request)

        // 401の場合トークンリフレッシュ
        if (response.code == 401) {
            response.close()
            val newToken = runBlocking { tokenManager.refreshToken() }
            val newRequest = chain.request().newBuilder()
                .addHeader("Authorization", "Bearer $newToken")
                .build()
            return chain.proceed(newRequest)
        }

        return response
    }
}
```

---

## リトライInterceptor

```kotlin
class RetryInterceptor(private val maxRetries: Int = 3) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var attempt = 0
        var lastException: IOException? = null

        while (attempt < maxRetries) {
            try {
                val response = chain.proceed(chain.request())
                if (response.isSuccessful || response.code !in listOf(500, 502, 503)) {
                    return response
                }
                response.close()
            } catch (e: IOException) {
                lastException = e
            }

            attempt++
            if (attempt < maxRetries) {
                Thread.sleep(1000L * attempt)  // 指数バックオフ
            }
        }

        throw lastException ?: IOException("Max retries exceeded")
    }
}
```

---

## OkHttpClient構築

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .addInterceptor(RetryInterceptor(maxRetries = 3))
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                else HttpLoggingInterceptor.Level.NONE
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
}
```

---

## まとめ

| Interceptor | 用途 |
|------------|------|
| 認証 | Authorization Header |
| ログ | HttpLoggingInterceptor |
| リトライ | 指数バックオフ |
| キャッシュ | Cache-Control |

- `Interceptor`でリクエスト/レスポンスを横断的に処理
- 認証トークンの自動付与とリフレッシュ
- リトライInterceptorで一時的なエラーに対応
- `HttpLoggingInterceptor`でデバッグ時のみログ出力

---

8種類のAndroidアプリテンプレート（ネットワーク設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [Certificate Pinning](https://zenn.dev/myougatheaxo/articles/android-compose-certificate-pinning-2026)
- [キャッシュ戦略](https://zenn.dev/myougatheaxo/articles/android-compose-caching-strategy-2026)
