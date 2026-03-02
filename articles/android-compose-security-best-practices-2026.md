---
title: "Androidセキュリティベストプラクティス — Compose/暗号化/証明書"
emoji: "🔒"
type: "tech"
topics: ["android", "kotlin", "security", "bestpractices"]
published: true
---

## この記事で学べること

Androidアプリの**セキュリティ対策**（暗号化、Network Security Config、EncryptedSharedPreferences）を解説します。

---

## EncryptedSharedPreferences

```kotlin
dependencies {
    implementation("androidx.security:security-crypto:1.1.0-alpha06")
}
```

```kotlin
fun getEncryptedPrefs(context: Context): SharedPreferences {
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

// 使用
val prefs = getEncryptedPrefs(context)
prefs.edit().putString("auth_token", token).apply()
val token = prefs.getString("auth_token", null)
```

---

## Network Security Config

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <!-- 本番: HTTPSのみ -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- デバッグ: ローカルHTTP許可 -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>

    <!-- 特定ドメインのみHTTP許可 -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

---

## Certificate Pinning

```kotlin
val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()
    )
    .build()
```

---

## 入力バリデーション

```kotlin
// SQLインジェクション防止: Room は自動でパラメータ化
@Query("SELECT * FROM users WHERE name = :name")
suspend fun findByName(name: String): User?

// XSS防止: WebViewでJavaScript無効化（不要な場合）
webView.settings.javaScriptEnabled = false

// パストラバーサル防止
fun safeFileName(input: String): String {
    return input.replace(Regex("[^a-zA-Z0-9._-]"), "_")
}

// Intent Data検証
fun handleDeepLink(intent: Intent) {
    val uri = intent.data ?: return
    val host = uri.host ?: return
    if (host !in listOf("example.com", "www.example.com")) {
        return // 不正なホスト
    }
}
```

---

## ProGuardでの難読化

```proguard
# リリースビルドで有効
-keep class com.example.app.data.model.** { *; }
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
# リフレクションを使うクラスは除外
```

---

## まとめ

- `EncryptedSharedPreferences`で機密データ暗号化
- Network Security Configで平文HTTP制限
- Certificate Pinningで中間者攻撃防止
- Room `@Query`でSQLインジェクション自動防止
- 入力は常にバリデーション（ファイル名、URL、パス）
- リリースビルドはProGuardで難読化必須

---

8種類のAndroidアプリテンプレート（セキュリティ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [生体認証ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-biometric-auth-2026)
- [パーミッション処理](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
