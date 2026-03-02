---
title: "OAuth2/PKCE完全ガイド — 認証フロー/トークン管理/リフレッシュ"
emoji: "🔑"
type: "tech"
topics: ["android", "kotlin", "oauth", "security"]
published: true
---

## この記事で学べること

**OAuth2/PKCE**（Authorization Code Flow + PKCE、トークン管理、自動リフレッシュ、セキュアストレージ）を解説します。

---

## PKCE フロー

```kotlin
class OAuthManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val tokenStorage: TokenStorage
) {
    private val codeVerifier = generateCodeVerifier()

    fun buildAuthUrl(): String {
        val codeChallenge = generateCodeChallenge(codeVerifier)

        return Uri.Builder()
            .scheme("https")
            .authority("auth.example.com")
            .appendPath("authorize")
            .appendQueryParameter("client_id", BuildConfig.OAUTH_CLIENT_ID)
            .appendQueryParameter("redirect_uri", "myapp://callback")
            .appendQueryParameter("response_type", "code")
            .appendQueryParameter("scope", "openid profile email")
            .appendQueryParameter("code_challenge", codeChallenge)
            .appendQueryParameter("code_challenge_method", "S256")
            .appendQueryParameter("state", generateState())
            .build()
            .toString()
    }

    private fun generateCodeVerifier(): String {
        val bytes = ByteArray(32)
        SecureRandom().nextBytes(bytes)
        return Base64.encodeToString(bytes, Base64.URL_SAFE or Base64.NO_PADDING or Base64.NO_WRAP)
    }

    private fun generateCodeChallenge(verifier: String): String {
        val digest = MessageDigest.getInstance("SHA-256").digest(verifier.toByteArray())
        return Base64.encodeToString(digest, Base64.URL_SAFE or Base64.NO_PADDING or Base64.NO_WRAP)
    }
}
```

---

## トークン交換

```kotlin
class TokenRepository @Inject constructor(
    private val api: AuthApi,
    private val tokenStorage: TokenStorage
) {
    suspend fun exchangeCode(code: String, codeVerifier: String): Result<AuthToken> {
        return try {
            val response = api.exchangeToken(
                grantType = "authorization_code",
                code = code,
                redirectUri = "myapp://callback",
                clientId = BuildConfig.OAUTH_CLIENT_ID,
                codeVerifier = codeVerifier
            )
            tokenStorage.save(response)
            Result.success(response)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    suspend fun refreshToken(): Result<AuthToken> {
        val refreshToken = tokenStorage.getRefreshToken() ?: return Result.failure(Exception("No refresh token"))

        return try {
            val response = api.refreshToken(
                grantType = "refresh_token",
                refreshToken = refreshToken,
                clientId = BuildConfig.OAUTH_CLIENT_ID
            )
            tokenStorage.save(response)
            Result.success(response)
        } catch (e: Exception) {
            tokenStorage.clear()
            Result.failure(e)
        }
    }
}
```

---

## 自動リフレッシュ Interceptor

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenStorage: TokenStorage,
    private val tokenRepository: Lazy<TokenRepository>
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = runBlocking { tokenStorage.getAccessToken() }

        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .build()

        val response = chain.proceed(request)

        if (response.code == 401) {
            response.close()
            val newToken = runBlocking { tokenRepository.get().refreshToken() }

            return newToken.fold(
                onSuccess = { authToken ->
                    chain.proceed(
                        chain.request().newBuilder()
                            .addHeader("Authorization", "Bearer ${authToken.accessToken}")
                            .build()
                    )
                },
                onFailure = { response }
            )
        }

        return response
    }
}
```

---

## セキュアストレージ

```kotlin
class TokenStorage @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val prefs = EncryptedSharedPreferences.create(
        context, "auth_tokens", masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun save(token: AuthToken) {
        prefs.edit()
            .putString("access_token", token.accessToken)
            .putString("refresh_token", token.refreshToken)
            .putLong("expires_at", System.currentTimeMillis() + token.expiresIn * 1000)
            .apply()
    }

    fun getAccessToken(): String? = prefs.getString("access_token", null)
    fun getRefreshToken(): String? = prefs.getString("refresh_token", null)
    fun clear() { prefs.edit().clear().apply() }
}
```

---

## まとめ

| 要素 | 実装 |
|------|------|
| PKCE | Code Verifier + Challenge |
| トークン保存 | EncryptedSharedPreferences |
| 自動リフレッシュ | OkHttp Interceptor |
| ログアウト | トークン削除 + リダイレクト |

- PKCE対応でモバイルアプリのOAuth2を安全に
- `EncryptedSharedPreferences`でトークンを暗号化保存
- Interceptorで401時の自動リフレッシュ
- `SecureRandom`で暗号学的に安全なcode_verifier生成

---

8種類のAndroidアプリテンプレート（認証機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Credential Manager](https://zenn.dev/myougatheaxo/articles/android-compose-credential-manager-2026)
- [証明書ピンニング](https://zenn.dev/myougatheaxo/articles/android-compose-certificate-pinning-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
