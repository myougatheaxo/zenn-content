---
title: "Retrofit Authenticator完全ガイド — トークンリフレッシュ/自動再認証/401処理"
emoji: "🔑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit Authenticator**（トークン自動リフレッシュ、401レスポンス処理、再認証フロー）を解説します。

---

## 基本Authenticator

```kotlin
class TokenAuthenticator @Inject constructor(
    private val tokenManager: TokenManager,
    private val authApi: Lazy<AuthApi>
) : Authenticator {

    override fun authenticate(route: Route?, response: Response): Request? {
        // 無限ループ防止
        if (response.request.header("X-Retry") != null) return null

        synchronized(this) {
            val newToken = runBlocking {
                val refreshToken = tokenManager.getRefreshToken() ?: return@runBlocking null
                try {
                    val result = authApi.get().refresh(RefreshRequest(refreshToken))
                    tokenManager.saveTokens(result.accessToken, result.refreshToken)
                    result.accessToken
                } catch (e: Exception) {
                    tokenManager.clearTokens()
                    null
                }
            } ?: return null

            return response.request.newBuilder()
                .header("Authorization", "Bearer $newToken")
                .header("X-Retry", "true")
                .build()
        }
    }
}
```

---

## TokenManager

```kotlin
class TokenManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val dataStore = context.dataStore

    suspend fun getAccessToken(): String? =
        dataStore.data.first()[stringPreferencesKey("access_token")]

    suspend fun getRefreshToken(): String? =
        dataStore.data.first()[stringPreferencesKey("refresh_token")]

    suspend fun saveTokens(access: String, refresh: String) {
        dataStore.edit {
            it[stringPreferencesKey("access_token")] = access
            it[stringPreferencesKey("refresh_token")] = refresh
        }
    }

    suspend fun clearTokens() {
        dataStore.edit { it.clear() }
    }
}
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
        tokenManager: TokenManager,
        authenticator: TokenAuthenticator
    ): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor { chain ->
            val token = runBlocking { tokenManager.getAccessToken() }
            val request = chain.request().newBuilder().apply {
                token?.let { header("Authorization", "Bearer $it") }
            }.build()
            chain.proceed(request)
        }
        .authenticator(authenticator)
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
}
```

---

## まとめ

| コンポーネント | 役割 |
|--------------|------|
| `Authenticator` | 401時の再認証 |
| `TokenManager` | トークン保存/取得 |
| Interceptor | リクエストにトークン付与 |
| `synchronized` | 並行リフレッシュ防止 |

- `Authenticator`は401レスポンス時に自動呼び出し
- `synchronized`で並行リフレッシュを防止
- `X-Retry`ヘッダーで無限ループ防止
- `Lazy<AuthApi>`で循環依存を回避

---

8種類のAndroidアプリテンプレート（API認証対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Retrofit Cache](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-cache-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
