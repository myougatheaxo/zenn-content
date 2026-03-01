---
title: "Gradleがわからなくても大丈夫 — AIが生成するAndroidプロジェクトのビルド設定解説"
emoji: "🔧"
type: "tech"
topics: ["Android", "Gradle", "Kotlin", "初心者", "AI"]
published: true
---

## Gradleって何？

Androidアプリを作ると必ず出てくる**ビルドツール**です。

「build.gradleが壊れた」「Sync Failedが消えない」——初心者が最もつまずくポイントの一つ。

でもAIが生成するプロジェクトでは、Gradleは**すでに正しく設定されている**ので、理解する必要はほぼありません。それでも「何をしているか」だけ知っておくと、トラブル時に助かります。

---

## AIが生成するbuild.gradle.kts

### プロジェクトレベル（ルート）

```kotlin
// build.gradle.kts (Project)
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
}
```

**やっていること**: Android Gradle PluginとKotlin Pluginを宣言するだけ。

### アプリレベル

```kotlin
// build.gradle.kts (Module: app)
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    id("com.google.devtools.ksp") version "2.0.0-1.0.21"
}

android {
    namespace = "com.example.habittracker"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.habittracker"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.10"
    }
}

dependencies {
    // Compose
    implementation(platform("androidx.compose:compose-bom:2024.02.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")

    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    // ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
}
```

---

## 5つだけ覚える設定項目

| 項目 | 意味 | 変えるタイミング |
|------|------|----------------|
| `applicationId` | Google Playでの一意識別子 | アプリ名を変えるとき |
| `minSdk` | サポートする最小Androidバージョン | 古い端末をサポートしたいとき |
| `targetSdk` | 最新APIへの準拠宣言 | Google Playの要件更新時 |
| `versionCode` | 内部バージョン番号（整数） | アップデート時に+1 |
| `versionName` | ユーザーに見えるバージョン名 | アップデート時 |

それ以外の設定は**AIが生成した状態のままでOK**です。

---

## よくあるトラブルと対処

### 「Sync Failed」

**原因**: ほぼ100%、依存関係のバージョン不整合。

**対処**: 
1. `File` → `Invalidate Caches / Restart`
2. それでもダメなら `./gradlew clean`
3. 最終手段: `.gradle` フォルダを削除してSync

### 「KSP version mismatch」

**原因**: KSP（Kotlin Symbol Processing）のバージョンがKotlinバージョンと不整合。

**対処**: AIに「KSPバージョンを修正して」と指示すれば直る。

### 「compileSdk does not match」

**原因**: compileSdkとtargetSdkが異なる。

**対処**: 両方を同じ値にする（通常は最新の34）。

---

## AIが生成するGradle設定の特徴

1. **Version Catalog（libs.versions.toml）** — バージョン管理が一箇所に集約
2. **KSP** — Room等のアノテーションプロセッサ用（kaptより高速）
3. **Compose BOM** — Composeライブラリのバージョンを一括管理
4. **最新のSDKバージョン** — compileSdk/targetSdkが最新

人間が手動で設定すると、バージョンの組み合わせで半日消耗することがあります。AIは**互換性のあるバージョンの組み合わせを知っている**ので、Sync Failedが起きません。

---

## まとめ

- Gradleはビルドツール。AIが設定済みなので基本触らなくていい
- 変えるのは `applicationId`, `versionCode`, `versionName` くらい
- Sync Failedは「キャッシュクリア」で大体直る
- AIは互換性のあるバージョン組み合わせを自動選択する

---

:::message
8種類のAndroidアプリテンプレート（Gradle設定済み、Sync確認済み）を[Gumroad](https://myougatheax.gumroad.com)で公開中。ビルドトラブルなし。
:::

---

## 関連記事

- [Claude Codeに「Androidアプリ作って」と言ったら47秒で完成した](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [コーディング経験ゼロからAndroidアプリを11分で動かす](https://zenn.dev/myougatheaxo/articles/android-app-no-code-guide)
- [Google Playに出す全手順](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [Kotlin Coroutineの4つだけ覚えればいいキーワード](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-android-basics)
- [CLAUDE.mdの書き方完全ガイド](https://zenn.dev/myougatheaxo/articles/claude-md-best-practices-2026)

:::message
Zenn Booksでも: [AIでAndroidアプリを作る実践ガイド](https://zenn.dev/myougatheaxo/books/ai-android-app-builder) (¥980)
:::
