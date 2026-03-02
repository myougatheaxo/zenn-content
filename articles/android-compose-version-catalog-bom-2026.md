---
title: "Version Catalog + BOM完全ガイド — 依存関係管理の決定版"
emoji: "📋"
type: "tech"
topics: ["android", "kotlin", "gradle", "dependency"]
published: true
---

## この記事で学べること

**Version Catalog**と**BOM**による依存関係管理（TOML定義、BOMの使い方、バージョン統一）を解説します。

---

## Version Catalog (libs.versions.toml)

```toml
# gradle/libs.versions.toml
[versions]
agp = "8.5.2"
kotlin = "2.0.21"
compose-bom = "2024.12.01"
hilt = "2.51.1"
room = "2.6.1"
ktor = "2.3.12"
coil = "2.7.0"
ksp = "2.0.21-1.0.27"

[libraries]
# Compose BOM
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
compose-navigation = { group = "androidx.navigation", name = "navigation-compose", version = "2.8.4" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version = "1.2.0" }

# Room
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }

# Ktor
ktor-client-core = { group = "io.ktor", name = "ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { group = "io.ktor", name = "ktor-client-okhttp", version.ref = "ktor" }
ktor-client-content-negotiation = { group = "io.ktor", name = "ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-json = { group = "io.ktor", name = "ktor-serialization-kotlinx-json", version.ref = "ktor" }

# Coil
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }

# Testing
junit = { group = "junit", name = "junit", version = "4.13.2" }
mockk = { group = "io.mockk", name = "mockk", version = "1.13.12" }
turbine = { group = "app.cash.turbine", name = "turbine", version = "1.1.0" }
coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version = "1.8.1" }

[bundles]
compose = ["compose-ui", "compose-material3", "compose-ui-tooling-preview"]
room = ["room-runtime", "room-ktx"]
ktor = ["ktor-client-core", "ktor-client-okhttp", "ktor-client-content-negotiation", "ktor-serialization-json"]
testing = ["junit", "mockk", "turbine", "coroutines-test"]

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

---

## build.gradle.kts（app）

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler)
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

dependencies {
    // Compose BOM
    val composeBom = platform(libs.compose.bom)
    implementation(composeBom)
    implementation(libs.bundles.compose)
    debugImplementation(libs.compose.ui.tooling)

    // Navigation
    implementation(libs.compose.navigation)

    // Hilt
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.hilt.navigation.compose)

    // Room
    implementation(libs.bundles.room)
    ksp(libs.room.compiler)

    // Ktor
    implementation(libs.bundles.ktor)

    // Coil
    implementation(libs.coil.compose)

    // Testing
    testImplementation(libs.bundles.testing)
}
```

---

## BOMの仕組み

```kotlin
// BOM = Bill of Materials
// バージョンを一括管理、個別指定不要

// ✅ BOM使用（バージョン自動解決）
val composeBom = platform(libs.compose.bom)
implementation(composeBom)
implementation("androidx.compose.ui:ui")           // バージョン不要
implementation("androidx.compose.material3:material3") // バージョン不要

// ❌ BOMなし（バージョン個別管理）
implementation("androidx.compose.ui:ui:1.7.5")
implementation("androidx.compose.material3:material3:1.3.1")
// バージョン不整合のリスク
```

---

## マルチモジュール対応

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}

// 全モジュールで libs.xxx が使える
// feature-home/build.gradle.kts
dependencies {
    implementation(libs.compose.material3)
    implementation(libs.hilt.android)
}
```

---

## バンドル活用

```kotlin
// libs.versions.toml の [bundles]
// 関連ライブラリをまとめて1行で追加

// compose = ["compose-ui", "compose-material3", "compose-ui-tooling-preview"]
implementation(libs.bundles.compose)

// room = ["room-runtime", "room-ktx"]
implementation(libs.bundles.room)

// testing = ["junit", "mockk", "turbine", "coroutines-test"]
testImplementation(libs.bundles.testing)
```

---

## まとめ

| 機能 | 説明 |
|------|------|
| `[versions]` | バージョン番号の一元管理 |
| `[libraries]` | ライブラリ座標の定義 |
| `[bundles]` | 関連ライブラリのグループ化 |
| `[plugins]` | Gradleプラグインの定義 |
| BOM | 同一プロジェクトのバージョン自動統一 |

- `version.ref`で`[versions]`を参照
- `alias(libs.plugins.xxx)`でプラグイン適用
- `libs.bundles.xxx`で複数ライブラリを1行追加
- `platform(libs.compose.bom)`でCompose BOM適用
- マルチモジュールでも全モジュールから`libs`参照可能

---

8種類のAndroidアプリテンプレート（Version Catalog設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [build.gradle.kts Tips](https://zenn.dev/myougatheaxo/articles/android-compose-build-gradle-tips-2026)
- [マルチモジュール](https://zenn.dev/myougatheaxo/articles/android-compose-multi-module-architecture-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
