---
title: "Androidアプリ署名完全ガイド — keystoreの作成からPlay App Signingまで"
emoji: "🔏"
type: "tech"
topics: ["android", "kotlin", "security", "googleplay"]
published: true
---

## この記事で学べること

Google Playにアプリを公開するには**署名**が必須です。keystoreの作成からPlay App Signingの設定まで、正しい手順を解説します。

---

## keystoreの作成

```bash
keytool -genkey -v \
  -keystore my-release-key.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias my-key-alias
```

| パラメータ | 説明 |
|-----------|------|
| `-keystore` | keystoreファイル名 |
| `-keyalg RSA` | 暗号アルゴリズム |
| `-keysize 2048` | 鍵の長さ |
| `-validity 10000` | 有効期間（日） |
| `-alias` | 鍵のエイリアス名 |

---

## build.gradle.ktsで署名設定

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file("../keystore/my-release-key.jks")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

**パスワードは環境変数から読み込む**のがベストプラクティス。build.gradle.ktsにハードコードしない。

---

## local.propertiesで開発用設定

```properties
# local.properties（.gitignoreに含める）
KEYSTORE_PASSWORD=your_password
KEY_ALIAS=my-key-alias
KEY_PASSWORD=your_key_password
```

```kotlin
// build.gradle.kts
val localProperties = Properties()
val localPropertiesFile = rootProject.file("local.properties")
if (localPropertiesFile.exists()) {
    localProperties.load(localPropertiesFile.inputStream())
}

signingConfigs {
    create("release") {
        storeFile = file("../keystore/my-release-key.jks")
        storePassword = localProperties["KEYSTORE_PASSWORD"] as? String
            ?: System.getenv("KEYSTORE_PASSWORD")
        keyAlias = localProperties["KEY_ALIAS"] as? String
            ?: System.getenv("KEY_ALIAS")
        keyPassword = localProperties["KEY_PASSWORD"] as? String
            ?: System.getenv("KEY_PASSWORD")
    }
}
```

---

## Play App Signing

Google Play App Signingを使えば、**Googleが署名鍵を安全に管理**します。

### メリット

| 項目 | 従来 | Play App Signing |
|------|------|-----------------|
| 鍵の紛失リスク | 自己責任 | Googleが管理 |
| 鍵の漏洩対策 | 自己責任 | Googleが保護 |
| AABサイズ最適化 | 不可 | 自動 |
| アップロード鍵のリセット | 不可 | 可能 |

### 設定方法

1. Google Play Consoleでアプリを作成
2. 「アプリの署名」セクションを開く
3. 「Play App Signingを使用」を選択
4. アップロード鍵のkeystoreを登録

---

## AABのビルド

```bash
# Android App Bundle（推奨）
./gradlew bundleRelease

# 出力先
# app/build/outputs/bundle/release/app-release.aab
```

Google PlayはAABを**必須**としています。APKではなくAABをアップロード。

---

## 署名の確認

```bash
# AABの署名を確認
jarsigner -verify -verbose app-release.aab

# keystoreの内容確認
keytool -list -v -keystore my-release-key.jks
```

---

## セキュリティチェックリスト

- [ ] keystoreファイルを`.gitignore`に追加
- [ ] パスワードをソースコードにハードコードしない
- [ ] keystoreのバックアップを安全な場所に保管
- [ ] Play App Signingを有効化
- [ ] CI/CDではSecrets（環境変数）で管理

---

## まとめ

- `keytool`でkeystoreを作成
- パスワードは環境変数 or local.propertiesで管理
- **Play App Signing**を有効化してGoogleに鍵管理を委託
- AAB形式でビルド・アップロード
- keystoreは`.gitignore`に含め、安全にバックアップ

---

8種類のAndroidアプリテンプレート（署名設定・リリースビルド構成済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [APKサイズ最適化](https://zenn.dev/myougatheaxo/articles/android-apk-optimization-2026)
- [Version Catalog入門](https://zenn.dev/myougatheaxo/articles/android-version-catalog-2026)
