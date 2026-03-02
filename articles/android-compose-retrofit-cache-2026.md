---
title: "Retrofit Cache完全ガイド — OkHttpキャッシュ/オフライン対応/ETag"
emoji: "💿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit Cache**（OkHttpキャッシュ、オフライン対応、ETag/Cache-Control、キャッシュ戦略）を解説します。

---

## 基本キャッシュ設定

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideCache(@ApplicationContext context: Context): Cache =
        Cache(File(context.cacheDir, "http_cache"), 10L * 1024 * 1024) // 10MB

    @Provides
    @Singleton
    fun provideOkHttpClient(cache: Cache): OkHttpClient =
        OkHttpClient.Builder()
            .cache(cache)
            .addInterceptor(CacheInterceptor())
            .addNetworkInterceptor(NetworkCacheInterceptor())
            .build()
}
```

---

## オフライン対応Interceptor

```kotlin
class CacheInterceptor @Inject constructor(
    @ApplicationContext private val context: Context
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var request = chain.request()

        if (!isNetworkAvailable(context)) {
            // オフライン時はキャッシュを強制使用
            request = request.newBuilder()
                .cacheControl(CacheControl.FORCE_CACHE)
                .build()
        }

        return chain.proceed(request)
    }

    private fun isNetworkAvailable(context: Context): Boolean {
        val cm = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        return cm.activeNetwork != null
    }
}

class NetworkCacheInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        val cacheControl = CacheControl.Builder()
            .maxAge(5, TimeUnit.MINUTES)
            .build()
        return response.newBuilder()
            .header("Cache-Control", cacheControl.toString())
            .removeHeader("Pragma")
            .build()
    }
}
```

---

## API別キャッシュ制御

```kotlin
interface ApiService {
    // キャッシュあり（デフォルト）
    @GET("articles")
    suspend fun getArticles(): List<Article>

    // キャッシュなし
    @Headers("Cache-Control: no-cache")
    @GET("notifications")
    suspend fun getNotifications(): List<Notification>

    // カスタムキャッシュ時間
    @Headers("Cache-Control: max-age=3600")
    @GET("categories")
    suspend fun getCategories(): List<Category>
}
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `Cache(dir, size)` | キャッシュディレクトリ/サイズ |
| `FORCE_CACHE` | オフライン時キャッシュ強制 |
| `max-age` | キャッシュ有効期間 |
| `no-cache` | キャッシュ無効化 |

- `OkHttpClient`に`Cache`を設定するだけで基本キャッシュ動作
- `CacheInterceptor`でオフライン時の自動キャッシュ利用
- `NetworkCacheInterceptor`でサーバーレスポンスのキャッシュ制御
- API別に`@Headers`でキャッシュポリシーを変更可能

---

8種類のAndroidアプリテンプレート（オフライン対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Retrofit Authenticator](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-authenticator-2026)
- [Retrofit Logging](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-logging-2026)
