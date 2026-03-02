---
title: "Version Catalog入門 — libs.versions.tomlで依存関係を一元管理"
emoji: "📦"
type: "tech"
topics: ["android", "gradle", "kotlin", "dependency"]
published: true
---

## この記事で学べること

Gradle Version Catalog（`libs.versions.toml`）を使えば、**プロジェクト全体の依存関係バージョンを1ファイルで管理**できます。

---

## libs.versions.toml

```toml
# gradle/libs.versions.toml

[versions]
kotlin = "2.0.0"
compose-bom = "2024.06.00"
room = "2.6.1"
hilt = "2.51"
retrofit = "2.11.0"
coil = "2.6.0"
coroutines = "1.8.1"

[libraries]
# Compose
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }

# Room
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

# Retrofit
retrofit-core = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }

# Coil
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }

# Coroutines
coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }

[bundles]
compose = ["compose-ui", "compose-material3", "compose-preview"]
room = ["room-runtime", "room-ktx"]
retrofit = ["retrofit-core", "retrofit-gson"]

[plugins]
android-application = { id = "com.android.application", version = "8.5.0" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
room = { id = "androidx.room", version.ref = "room" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

---

## build.gradle.ktsでの使い方

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler)
}

dependencies {
    // BOM
    implementation(platform(libs.compose.bom))

    // Bundle（複数ライブラリをまとめて）
    implementation(libs.bundles.compose)
    implementation(libs.bundles.room)
    implementation(libs.bundles.retrofit)

    // 個別
    implementation(libs.coil.compose)
    implementation(libs.coroutines.android)

    // KSP（コード生成）
    ksp(libs.room.compiler)
}
```

---

## bundlesの活用

```toml
[bundles]
compose = ["compose-ui", "compose-material3", "compose-preview"]
room = ["room-runtime", "room-ktx"]
```

```kotlin
// 1行で複数ライブラリを追加
implementation(libs.bundles.compose)
```

関連するライブラリを**bundle**にまとめると、build.gradle.ktsがすっきりします。

---

## バージョン更新

```toml
# 変更前
[versions]
room = "2.6.1"

# 変更後（この1箇所だけ）
[versions]
room = "2.7.0"
```

バージョン番号が1ファイルに集約されるため、**更新漏れがなくなります**。

---

## マルチモジュール対応

```
project/
├── gradle/
│   └── libs.versions.toml  ← 全モジュール共通
├── app/
│   └── build.gradle.kts
├── feature/
│   └── build.gradle.kts    ← 同じlibs.XXXが使える
└── core/
    └── build.gradle.kts    ← 同じlibs.XXXが使える
```

全モジュールで同じ`libs`を参照できるため、バージョン不一致が起きません。

---

## 従来方式との比較

| 方式 | メリット | デメリット |
|------|---------|-----------|
| `ext { }` | 柔軟 | 型安全でない |
| `buildSrc` | Kotlin DSP | ビルド時間増加 |
| **Version Catalog** | 型安全・IDE補完・標準 | 学習コスト（小） |

Version Catalogが**現在のGradle標準**です。

---

## まとめ

- `gradle/libs.versions.toml`に全バージョンを集約
- `libs.XXX`で型安全にアクセス（IDE補完付き）
- `bundles`で関連ライブラリをグループ化
- マルチモジュールでもバージョン統一
- Gradle 7.0+で利用可能（Android Studio推奨）

---

8種類のAndroidアプリテンプレート（Version Catalog対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [MVVM Architecture完全ガイド](https://zenn.dev/myougatheaxo/articles/android-mvvm-architecture-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
- [Room Migration完全ガイド](https://zenn.dev/myougatheaxo/articles/room-migration-guide-2026)
