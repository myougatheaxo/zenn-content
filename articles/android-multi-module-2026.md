---
title: "Androidマルチモジュール入門 — 大規模アプリの設計パターン"
emoji: "🏗️"
type: "tech"
topics: ["android", "kotlin", "architecture", "gradle"]
published: true
---

## この記事で学べること

アプリが大きくなると1モジュールでは限界が来ます。**マルチモジュール設計**でビルド時間短縮・チーム開発効率化・テスト容易化を実現する方法を解説します。

---

## なぜマルチモジュール？

| 課題 | マルチモジュールで解決 |
|------|---------------------|
| ビルドが遅い | 変更したモジュールだけ再ビルド |
| 依存関係が複雑 | モジュール間の依存を明確化 |
| テストが難しい | モジュール単位でテスト可能 |
| チーム開発で衝突 | モジュールごとに担当分け |

---

## 基本構成

```
project/
├── app/                    # エントリポイント
├── core/
│   ├── data/              # Repository, DataSource
│   ├── database/          # Room DB, DAO
│   ├── network/           # Retrofit, API
│   ├── model/             # データクラス
│   └── ui/                # 共通UIコンポーネント
├── feature/
│   ├── home/              # ホーム画面
│   ├── settings/          # 設定画面
│   └── detail/            # 詳細画面
└── gradle/
    └── libs.versions.toml
```

---

## モジュールの依存関係

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(project(":core:data"))
    implementation(project(":core:ui"))
    implementation(project(":feature:home"))
    implementation(project(":feature:settings"))
    implementation(project(":feature:detail"))
}

// feature/home/build.gradle.kts
dependencies {
    implementation(project(":core:data"))
    implementation(project(":core:model"))
    implementation(project(":core:ui"))
}

// core/data/build.gradle.kts
dependencies {
    implementation(project(":core:database"))
    implementation(project(":core:network"))
    implementation(project(":core:model"))
}
```

---

## 依存の方向

```
app → feature/* → core/data → core/database
                            → core/network
                → core/ui
                → core/model
```

**原則**: 上位モジュールが下位モジュールに依存。逆方向の依存は禁止。

---

## settings.gradle.kts

```kotlin
// settings.gradle.kts
include(":app")
include(":core:data")
include(":core:database")
include(":core:network")
include(":core:model")
include(":core:ui")
include(":feature:home")
include(":feature:settings")
include(":feature:detail")
```

---

## Convention Plugins（ビルド設定の共通化）

```kotlin
// build-logic/convention/src/main/kotlin/AndroidLibraryPlugin.kt
class AndroidLibraryPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("com.android.library")
            pluginManager.apply("org.jetbrains.kotlin.android")

            extensions.configure<LibraryExtension> {
                compileSdk = 34
                defaultConfig.minSdk = 26
                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }
            }
        }
    }
}
```

```kotlin
// feature/home/build.gradle.kts
plugins {
    id("myapp.android.library")
    id("myapp.android.compose")
}
```

---

## モジュール間のナビゲーション

```kotlin
// core/ui に共通ルート定義
object Routes {
    const val HOME = "home"
    const val SETTINGS = "settings"
    fun detail(id: Int) = "detail/$id"
}

// app のNavHost
NavHost(navController, startDestination = Routes.HOME) {
    homeScreen(navController)      // feature/home が提供
    settingsScreen(navController)  // feature/settings が提供
    detailScreen(navController)    // feature/detail が提供
}
```

---

## 小規模アプリは不要

| アプリ規模 | 推奨 |
|-----------|------|
| 画面5個以下 | シングルモジュール |
| 画面5〜15個 | 2〜3モジュール |
| 画面15個以上 | フルマルチモジュール |

**テンプレートアプリ**（画面2〜4個）ならシングルモジュールで十分です。

---

## まとめ

- マルチモジュールで**ビルド時間短縮・テスト容易化**
- `app` → `feature/*` → `core/*` の依存方向
- Convention Pluginsでビルド設定を共通化
- Version Catalogで依存関係を一元管理
- 小規模アプリには不要（過剰設計に注意）

---

8種類のAndroidアプリテンプレート（クリーンなシングルモジュール設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [MVVM Architecture完全ガイド](https://zenn.dev/myougatheaxo/articles/android-mvvm-architecture-2026)
- [Version Catalog入門](https://zenn.dev/myougatheaxo/articles/android-version-catalog-2026)
- [DI — Manual vs Hilt vs Koin](https://zenn.dev/myougatheaxo/articles/android-di-manual-vs-hilt-2026)
