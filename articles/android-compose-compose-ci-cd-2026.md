---
title: "Compose CI/CD完全ガイド — GitHub Actions/自動ビルド/テスト/リリース"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "cicd"]
published: true
---

## この記事で学べること

**Compose CI/CD**（GitHub Actions、自動ビルド・テスト、APK/AAB生成、Google Playデプロイ）を解説します。

---

## GitHub Actions基本

```yaml
# .github/workflows/android.yml
name: Android CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: Run lint
        run: ./gradlew lint

      - name: Run unit tests
        run: ./gradlew testDebugUnitTest

      - name: Build debug APK
        run: ./gradlew assembleDebug

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: debug-apk
          path: app/build/outputs/apk/debug/app-debug.apk
```

---

## リリースビルド

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: [ 'v*' ]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Decode Keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > app/release.keystore

      - name: Build Release AAB
        run: ./gradlew bundleRelease
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}

      - name: Upload to Google Play
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT }}
          packageName: com.example.app
          releaseFiles: app/build/outputs/bundle/release/app-release.aab
          track: production
          status: completed
```

---

## Signing設定

```kotlin
// build.gradle.kts
android {
    signingConfigs {
        create("release") {
            storeFile = file("release.keystore")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro")
        }
    }
}
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `actions/setup-java` | JDK設定 |
| `gradle/actions/setup-gradle` | Gradleキャッシュ |
| `upload-google-play` | Play Store公開 |
| `secrets` | 署名鍵管理 |

- GitHub ActionsでPR時にlint+テスト自動実行
- タグプッシュでリリースAAB生成+Play Storeデプロイ
- 署名鍵はBase64エンコードしてSecretsに保存
- Gradleキャッシュでビルド時間を短縮

---

8種類のAndroidアプリテンプレート（CI/CD設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose R8最適化](https://zenn.dev/myougatheaxo/articles/android-compose-compose-r8-optimization-2026)
- [Compose ProGuard](https://zenn.dev/myougatheaxo/articles/android-compose-compose-proguard-2026)
- [Gradle VersionCatalog](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-version-catalog-2026)
