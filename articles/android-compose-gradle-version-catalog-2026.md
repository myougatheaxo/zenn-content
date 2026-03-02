---
title: "Gradle Version Catalog完全ガイド — libs.versions.toml/バージョン一元管理"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gradle"]
published: true
---

## この記事で学べること

**Gradle Version Catalog**（libs.versions.toml、バージョン一元管理、BOM、プラグイン管理）を解説します。

---

## libs.versions.toml

```toml
# gradle/libs.versions.toml
[versions]
agp = "8.5.0"
kotlin = "2.0.0"
compose-bom = "2024.06.00"
hilt = "2.51.1"
room = "2.6.1"
retrofit = "2.11.0"
coroutines = "1.8.1"
lifecycle = "2.8.3"
navigation = "2.8.0"

[libraries]
# Compose BOM
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }

# Lifecycle
lifecycle-runtime = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }
lifecycle-viewmodel = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
hilt-navigation = { group = "androidx.hilt", name = "hilt-navigation-compose", version = "1.2.0" }

# Room
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }

# Retrofit
retrofit-core = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }

[bundles]
compose = ["compose-ui", "compose-material3", "compose-ui-tooling-preview"]
room = ["room-runtime", "room-ktx"]

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version = "2.0.0-1.0.22" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
```

---

## build.gradleでの使用

```kotlin
// build.gradle.kts (app)
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler)
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

dependencies {
    // BOM
    implementation(platform(libs.compose.bom))

    // Bundle（複数ライブラリをまとめて）
    implementation(libs.bundles.compose)
    implementation(libs.bundles.room)

    // 個別
    implementation(libs.lifecycle.runtime)
    implementation(libs.lifecycle.viewmodel)
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    ksp(libs.room.compiler)
    implementation(libs.hilt.navigation)
    implementation(libs.retrofit.core)
    implementation(libs.retrofit.gson)

    // Debug
    debugImplementation(libs.compose.ui.tooling)
}
```

---

## まとめ

| セクション | 用途 |
|-----------|------|
| `[versions]` | バージョン番号定義 |
| `[libraries]` | ライブラリ座標 |
| `[bundles]` | ライブラリグループ |
| `[plugins]` | Gradleプラグイン |

- `libs.versions.toml`で全依存関係を一元管理
- `version.ref`で同じバージョンを複数ライブラリで共有
- `bundles`で関連ライブラリをグループ化
- IDEの補完が効き、タイポを防止

---

8種類のAndroidアプリテンプレート（Version Catalog対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MultiModule](https://zenn.dev/myougatheaxo/articles/android-compose-compose-multi-module-2026)
- [Compose CI/CD](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ci-cd-2026)
- [Compose R8最適化](https://zenn.dev/myougatheaxo/articles/android-compose-compose-r8-optimization-2026)
