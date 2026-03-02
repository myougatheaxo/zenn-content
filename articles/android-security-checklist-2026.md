---
title: "Androidアプリセキュリティチェックリスト — リリース前に確認すべき15項目"
emoji: "🔒"
type: "tech"
topics: ["android", "kotlin", "security", "googleplay"]
published: true
---

## この記事で学べること

Google Playに公開する前に確認すべき**セキュリティチェックリスト15項目**を解説します。ユーザーデータを守り、ストアの審査を通過しましょう。

---

## データ保存

### 1. SharedPreferencesに機密データを保存しない

```kotlin
// ❌ 平文で保存（root端末で読める）
prefs.edit().putString("api_key", "sk-12345").apply()

// ✅ EncryptedSharedPreferences
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val encryptedPrefs = EncryptedSharedPreferences.create(
    context, "secret_prefs", masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

### 2. DataStoreも暗号化を検討

```kotlin
// 機密データにはEncryptedSharedPreferencesを使い、
// 通常の設定にはDataStoreを使う
```

### 3. ログに機密情報を出さない

```kotlin
// ❌
Log.d("Auth", "Token: $accessToken")

// ✅ リリースビルドではログ無効化
if (BuildConfig.DEBUG) {
    Log.d("Auth", "Token refreshed")
}
```

---

## ネットワーク

### 4. HTTPS必須

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
</network-security-config>
```

### 5. Certificate Pinning

```kotlin
val okHttpClient = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            .add("api.example.com", "sha256/XXXX...")
            .build()
    )
    .build()
```

### 6. APIキーをコードに埋め込まない

```kotlin
// ❌ ソースコードにハードコード
val API_KEY = "sk-12345"

// ✅ local.propertiesから読み込み
val API_KEY = BuildConfig.API_KEY
```

```kotlin
// build.gradle.kts
val localProps = Properties()
localProps.load(rootProject.file("local.properties").inputStream())

android {
    defaultConfig {
        buildConfigField("String", "API_KEY",
            "\"${localProps["API_KEY"]}\"")
    }
}
```

---

## アプリ保護

### 7. R8/ProGuardで難読化

```kotlin
release {
    isMinifyEnabled = true
    isShrinkResources = true
}
```

### 8. デバッグ無効化

```xml
<!-- リリースビルド -->
<application android:debuggable="false">
```

### 9. root検出

```kotlin
fun isRooted(): Boolean {
    val paths = arrayOf("/system/app/Superuser.apk", "/system/xbin/su")
    return paths.any { File(it).exists() }
}
```

---

## 入力検証

### 10. ユーザー入力のサニタイズ

```kotlin
// SQLインジェクション防止 → Room の @Query パラメータを使う
@Query("SELECT * FROM users WHERE name = :name")
fun findByName(name: String): User?

// WebViewのJavaScript無効化
webView.settings.javaScriptEnabled = false
```

### 11. Intent検証

```kotlin
// 外部からのIntentを検証
val data = intent.data
if (data != null && data.scheme == "https" && data.host == "example.com") {
    // 安全なデータのみ処理
}
```

---

## 権限

### 12. 最小権限の原則

必要なパーミッションだけ宣言。不要になったパーミッションは即削除。

### 13. Exported コンポーネントの制限

```xml
<!-- 外部からアクセス不要なActivityはexported=false -->
<activity android:name=".SettingsActivity" android:exported="false" />
```

---

## リリース

### 14. Play App Signingを有効化

Google が署名鍵を安全に管理。

### 15. プライバシーポリシーの作成

Google Playの要件。個人データの収集・使用方法を明記。

---

## まとめ

| カテゴリ | チェック項目 |
|---------|------------|
| データ保存 | 暗号化、ログ無効化 |
| ネットワーク | HTTPS必須、Pinning、APIキー管理 |
| アプリ保護 | R8、デバッグ無効化、root検出 |
| 入力検証 | SQL防止、Intent検証 |
| 権限 | 最小権限、exported制限 |
| リリース | App Signing、プライバシーポリシー |

---

8種類のAndroidアプリテンプレート（セキュリティベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ProGuard/R8完全ガイド](https://zenn.dev/myougatheaxo/articles/android-proguard-r8-2026)
- [アプリ署名完全ガイド](https://zenn.dev/myougatheaxo/articles/android-signing-release-2026)
- [パーミッション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-permission-runtime-2026)
