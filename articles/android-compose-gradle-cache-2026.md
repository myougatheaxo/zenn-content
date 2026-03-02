---
title: "Gradleビルド高速化完全ガイド — キャッシュ/並列ビルド/設定最適化"
emoji: "🏎️"
type: "tech"
topics: ["android", "kotlin", "gradle", "performance"]
published: true
---

## この記事で学べること

**Gradleビルド高速化**（ビルドキャッシュ、並列ビルド、Configuration Cache、JVM設定、Build Scan）を解説します。

---

## gradle.properties 最適化

```properties
# メモリ設定
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC -XX:MaxMetaspaceSize=1g

# 並列ビルド
org.gradle.parallel=true

# ビルドキャッシュ
org.gradle.caching=true

# Configuration Cache (Gradle 8.1+)
org.gradle.configuration-cache=true

# Kotlin デーモン
kotlin.daemon.jvmargs=-Xmx2g

# 非推移的R クラス（Android）
android.nonTransitiveRClass=true

# Kotlin インクリメンタル
kotlin.incremental=true

# KSP インクリメンタル
ksp.incremental=true
```

---

## ビルドスキャン

```kotlin
// settings.gradle.kts
plugins {
    id("com.gradle.develocity") version "3.17"
}

develocity {
    buildScan {
        termsOfUseUrl.set("https://gradle.com/help/legal-terms-of-use")
        termsOfUseAgree.set("yes")
        publishing.onlyIf { true } // 常にアップロード
    }
}
```

```bash
# ビルドスキャン実行
./gradlew assembleDebug --scan

# ビルド時間の分析
./gradlew assembleDebug --profile
# → build/reports/profile/ にレポート生成
```

---

## タスク設定の最適化

```kotlin
// build.gradle.kts (app)
android {
    // デバッグビルドでリソース圧縮無効
    buildTypes {
        debug {
            isMinifyEnabled = false
            isShrinkResources = false
            // PNG crunchを無効化
            aaptOptions.cruncherEnabled = false
        }
    }

    // 不要なビルドバリアントを無効化
    buildFeatures {
        buildConfig = true
        // 使わない機能を無効化
        aidl = false
        renderScript = false
    }

    // テスト設定
    testOptions {
        unitTests.isReturnDefaultValues = true
    }
}

// Kotlin コンパイラオプション
kotlin {
    compilerOptions {
        // デバッグビルドで最適化スキップ
        if (project.hasProperty("debug")) {
            freeCompilerArgs.addAll("-Xskip-prerelease-check")
        }
    }
}
```

---

## CI/CDキャッシュ

```yaml
# .github/workflows/build.yml
- name: Cache Gradle
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
      ~/.konan
    key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: gradle-

- name: Build
  run: ./gradlew assembleDebug --build-cache --no-daemon
```

---

## まとめ

| 設定 | 効果 |
|------|------|
| `parallel=true` | モジュール並列ビルド |
| `caching=true` | ビルドキャッシュ |
| `configuration-cache` | 設定フェーズのキャッシュ |
| `Xmx4g` | ヒープメモリ増加 |

- `gradle.properties`の最適化で30-50%高速化
- ビルドスキャンでボトルネックを特定
- デバッグビルドでリソース処理を最小化
- CI/CDでGradleキャッシュを活用

---

8種類のAndroidアプリテンプレート（ビルド最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Gradle Convention Plugin](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-convention-plugin-2026)
- [CI/CD](https://zenn.dev/myougatheaxo/articles/android-compose-ci-cd-github-actions-2026)
- [マルチモジュール](https://zenn.dev/myougatheaxo/articles/android-compose-multi-module-2026)
