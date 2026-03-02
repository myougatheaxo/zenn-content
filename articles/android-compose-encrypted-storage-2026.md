---
title: "暗号化ストレージ完全ガイド — EncryptedSharedPreferences/Keystore"
emoji: "🔐"
type: "tech"
topics: ["android", "kotlin", "security", "encryption"]
published: true
---

## この記事で学べること

Androidの**暗号化ストレージ**（EncryptedSharedPreferences、AndroidKeyStore、暗号化DataStore）を解説します。

---

## EncryptedSharedPreferences

```kotlin
// build.gradle.kts
// implementation("androidx.security:security-crypto:1.1.0-alpha06")

fun createEncryptedPrefs(context: Context): SharedPreferences {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    return EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
}

// 使用（通常のSharedPreferencesと同じAPI）
val prefs = createEncryptedPrefs(context)

// 書き込み
prefs.edit().apply {
    putString("auth_token", "eyJhbGciOi...")
    putString("refresh_token", "dGhpcyBpcyBh...")
    putLong("token_expiry", System.currentTimeMillis() + 3600000)
    apply()
}

// 読み取り
val token = prefs.getString("auth_token", null)
```

---

## AndroidKeyStore

```kotlin
object KeyStoreHelper {
    private const val ALIAS = "app_secret_key"

    fun getOrCreateKey(): SecretKey {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }

        if (keyStore.containsAlias(ALIAS)) {
            return (keyStore.getEntry(ALIAS, null) as KeyStore.SecretKeyEntry).secretKey
        }

        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
        )
        keyGenerator.init(
            KeyGenParameterSpec.Builder(ALIAS,
                KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setKeySize(256)
                .build()
        )
        return keyGenerator.generateKey()
    }

    fun encrypt(data: String): Pair<ByteArray, ByteArray> {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, getOrCreateKey())
        val encrypted = cipher.doFinal(data.toByteArray())
        return cipher.iv to encrypted
    }

    fun decrypt(iv: ByteArray, encryptedData: ByteArray): String {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        val spec = GCMParameterSpec(128, iv)
        cipher.init(Cipher.DECRYPT_MODE, getOrCreateKey(), spec)
        return String(cipher.doFinal(encryptedData))
    }
}
```

---

## セキュアなトークン管理

```kotlin
class SecureTokenManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val prefs by lazy { createEncryptedPrefs(context) }

    var accessToken: String?
        get() = prefs.getString("access_token", null)
        set(value) = prefs.edit().putString("access_token", value).apply()

    var refreshToken: String?
        get() = prefs.getString("refresh_token", null)
        set(value) = prefs.edit().putString("refresh_token", value).apply()

    val isLoggedIn: Boolean
        get() = accessToken != null

    fun clearTokens() {
        prefs.edit().clear().apply()
    }
}

// Hilt Module
@Module
@InstallIn(SingletonComponent::class)
object SecurityModule {
    @Provides
    @Singleton
    fun provideTokenManager(@ApplicationContext context: Context): SecureTokenManager {
        return SecureTokenManager(context)
    }
}
```

---

## OkHttpインターセプター連携

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenManager: SecureTokenManager
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder().apply {
            tokenManager.accessToken?.let {
                addHeader("Authorization", "Bearer $it")
            }
        }.build()
        return chain.proceed(request)
    }
}
```

---

## まとめ

| 方法 | 用途 | セキュリティ |
|------|------|------------|
| EncryptedSharedPreferences | トークン、設定 | AES-256暗号化 |
| AndroidKeyStore | 暗号鍵の安全な保管 | ハードウェアバック |
| DataStore + 暗号化 | 構造化データ | カスタム実装 |

- `MasterKey.KeyScheme.AES256_GCM`で鍵生成
- トークンは必ず暗号化ストレージに保存
- `clearTokens()`でログアウト時にクリア
- `AuthInterceptor`で自動的にヘッダー付与

---

8種類のAndroidアプリテンプレート（セキュリティ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [セキュリティベストプラクティス](https://zenn.dev/myougatheaxo/articles/android-compose-security-best-practices-2026)
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [生体認証](https://zenn.dev/myougatheaxo/articles/android-compose-biometric-auth-2026)
