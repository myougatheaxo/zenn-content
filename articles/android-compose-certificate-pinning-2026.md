---
title: "証明書ピンニング完全ガイド — OkHttp/Network Security Config/動的更新"
emoji: "🔐"
type: "tech"
topics: ["android", "kotlin", "security", "network"]
published: true
---

## この記事で学べること

**証明書ピンニング**（OkHttp CertificatePinner、Network Security Config、動的ピン更新、デバッグ設定）を解説します。

---

## OkHttp CertificatePinner

```kotlin
@Provides
@Singleton
fun provideOkHttpClient(): OkHttpClient {
    val certificatePinner = CertificatePinner.Builder()
        .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
        .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=") // バックアップ
        .add("*.example.com", "sha256/CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=")
        .build()

    return OkHttpClient.Builder()
        .certificatePinner(certificatePinner)
        .build()
}

// ピンのハッシュ取得方法:
// openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout |
//   openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl enc -base64
```

---

## Network Security Config

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <!-- 本番用ピンニング -->
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>

    <!-- デバッグ時はユーザー追加CA許可 -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
            <certificates src="system" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

---

## デバッグ/リリース切り替え

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().apply {
            if (!BuildConfig.DEBUG) {
                // リリースビルドのみピンニング有効
                certificatePinner(
                    CertificatePinner.Builder()
                        .add("api.example.com", "sha256/...")
                        .add("api.example.com", "sha256/...") // バックアップピン
                        .build()
                )
            }
        }.build()
    }
}
```

---

## ピン失敗のハンドリング

```kotlin
class SafeApiClient @Inject constructor(
    private val okHttpClient: OkHttpClient
) {
    suspend fun <T> safeCall(block: suspend () -> T): Result<T> {
        return try {
            Result.success(block())
        } catch (e: SSLPeerUnverifiedException) {
            // 証明書ピンニング失敗
            // → 中間者攻撃の可能性
            Log.e("Security", "Certificate pinning failed", e)
            Result.failure(SecurityException("通信の安全性を確認できません"))
        } catch (e: SSLHandshakeException) {
            Log.e("Security", "SSL handshake failed", e)
            Result.failure(SecurityException("安全な接続を確立できません"))
        }
    }
}
```

---

## まとめ

| 手法 | 特徴 |
|------|------|
| `CertificatePinner` | コードで設定、柔軟 |
| Network Security Config | XML宣言的、有効期限 |
| バックアップピン | ローテーション対応 |
| デバッグ切替 | 開発時はピンニング無効 |

- バックアップピンを必ず設定（証明書ローテーション対応）
- `expiration`で有効期限を設定（期限切れ時は自動無効化）
- デバッグビルドではCharles Proxy等が使えるよう無効化
- `SSLPeerUnverifiedException`で中間者攻撃検知

---

8種類のAndroidアプリテンプレート（セキュリティ対策済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [Firebase App Check](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-appcheck-2026)
- [Credential Manager](https://zenn.dev/myougatheaxo/articles/android-compose-credential-manager-2026)
