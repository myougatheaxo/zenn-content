---
title: "GitHub Actions CI/CD完全ガイド — Androidビルド自動化"
emoji: "🏗️"
type: "tech"
topics: ["android", "kotlin", "githubactions", "cicd"]
published: true
---

## この記事で学べること

**GitHub Actions**でAndroidアプリの**CI/CD**（ビルド、テスト、Lint、リリース自動化）を構築する方法を解説します。

---

## 基本ワークフロー

```yaml
# .github/workflows/android-ci.yml
name: Android CI

on:
  push:
    branches: [ main, develop ]
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
        uses: gradle/actions/setup-gradle@v4

      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}

      - name: Build Debug
        run: ./gradlew assembleDebug

      - name: Run Unit Tests
        run: ./gradlew testDebugUnitTest

      - name: Run Lint
        run: ./gradlew lintDebug

      - name: Upload Lint Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: app/build/reports/lint-results-debug.html
```

---

## テスト + カバレッジ

```yaml
  test:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Run Tests with Coverage
        run: ./gradlew testDebugUnitTest jacocoTestReport

      - name: Upload Coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: app/build/reports/jacoco/

      - name: Check Coverage Threshold
        run: |
          COVERAGE=$(cat app/build/reports/jacoco/test/html/index.html | grep -o 'Total[^%]*%' | grep -o '[0-9]*%' | head -1)
          echo "Coverage: $COVERAGE"
```

---

## Instrumented Test（エミュレーター）

```yaml
  instrumented-test:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Enable KVM
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm

      - name: Instrumented Tests
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          arch: x86_64
          script: ./gradlew connectedDebugAndroidTest
```

---

## Release ビルド + 署名

```yaml
  release:
    runs-on: ubuntu-latest
    needs: [ build, test ]
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Decode Keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > app/release.keystore

      - name: Build Release APK
        run: ./gradlew assembleRelease
        env:
          SIGNING_KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          SIGNING_KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          SIGNING_STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }}

      - name: Build Release AAB
        run: ./gradlew bundleRelease
        env:
          SIGNING_KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          SIGNING_KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          SIGNING_STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }}

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: release-apk
          path: app/build/outputs/apk/release/*.apk

      - name: Upload AAB
        uses: actions/upload-artifact@v4
        with:
          name: release-aab
          path: app/build/outputs/bundle/release/*.aab
```

---

## build.gradle.kts 署名設定

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file("release.keystore")
            storePassword = System.getenv("SIGNING_STORE_PASSWORD")
            keyAlias = System.getenv("SIGNING_KEY_ALIAS")
            keyPassword = System.getenv("SIGNING_KEY_PASSWORD")
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

---

## まとめ

| ジョブ | 内容 |
|--------|------|
| build | assembleDebug + lint |
| test | Unit Test + カバレッジ |
| instrumented | エミュレーター UIテスト |
| release | 署名付きAPK/AAB |

- `actions/cache`でGradleキャッシュ高速化
- `secrets`で署名鍵を安全に管理
- `needs`でジョブ依存関係を定義
- PRマージ時のみリリースビルド実行

---

8種類のAndroidアプリテンプレート（CI/CD設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-baseline-profile-2026)
- [Version Catalog](https://zenn.dev/myougatheaxo/articles/android-compose-version-catalog-bom-2026)
- [統合テスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-integration-2026)
