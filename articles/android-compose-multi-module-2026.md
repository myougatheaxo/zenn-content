---
title: "マルチモジュール設計ガイド — レイヤー分割/Convention Plugin"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "gradle", "architecture"]
published: true
---

## この記事で学べること

**マルチモジュール設計**（レイヤーモジュール分割、Convention Plugin、依存関係管理、ビルド高速化）を解説します。

---

## モジュール構成

```
project/
├── app/                    # Application module
├── core/
│   ├── core-data/         # Repository, DataSource
│   ├── core-domain/       # UseCase, Entity
│   ├── core-network/      # Retrofit, API定義
│   ├── core-database/     # Room, DAO
│   ├── core-ui/           # 共通Composable
│   └── core-common/       # ユーティリティ
├── feature/
│   ├── feature-home/      # ホーム画面
│   ├── feature-detail/    # 詳細画面
│   ├── feature-settings/  # 設定画面
│   └── feature-auth/      # 認証画面
└── build-logic/           # Convention Plugins
    └── convention/
```

---

## settings.gradle.kts

```kotlin
// settings.gradle.kts
pluginManagement {
    includeBuild("build-logic")
}

include(":app")
include(":core:core-data")
include(":core:core-domain")
include(":core:core-network")
include(":core:core-database")
include(":core:core-ui")
include(":core:core-common")
include(":feature:feature-home")
include(":feature:feature-detail")
include(":feature:feature-settings")
include(":feature:feature-auth")
```

---

## Convention Plugin

```kotlin
// build-logic/convention/build.gradle.kts
plugins {
    `kotlin-dsl`
}

dependencies {
    compileOnly("com.android.tools.build:gradle:8.7.3")
    compileOnly("org.jetbrains.kotlin:kotlin-gradle-plugin:2.1.0")
}

// build-logic/convention/src/main/kotlin/AndroidLibraryConventionPlugin.kt
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<LibraryExtension> {
                compileSdk = 35
                defaultConfig {
                    minSdk = 26
                    testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                }
                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }
            }
        }
    }
}

// build-logic/convention/src/main/kotlin/AndroidFeatureConventionPlugin.kt
class AndroidFeatureConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("myapp.android.library")
            pluginManager.apply("myapp.android.hilt")
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")

            dependencies {
                add("implementation", project(":core:core-ui"))
                add("implementation", project(":core:core-domain"))
            }
        }
    }
}
```

```kotlin
// build-logic/convention/build.gradle.kts の gradlePlugin 登録
gradlePlugin {
    plugins {
        register("androidLibrary") {
            id = "myapp.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidFeature") {
            id = "myapp.android.feature"
            implementationClass = "AndroidFeatureConventionPlugin"
        }
    }
}
```

---

## Feature モジュール

```kotlin
// feature/feature-home/build.gradle.kts
plugins {
    id("myapp.android.feature") // Convention Plugin適用
}

dependencies {
    implementation(project(":core:core-data"))
}

// feature/feature-home/src/main/java/HomeScreen.kt
@Composable
fun HomeScreen(
    onItemClick: (String) -> Unit,
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // UI実装
}
```

---

## 依存関係ルール

```kotlin
// app → feature-*, core-*
// feature-* → core-domain, core-ui, core-data
// core-data → core-domain, core-network, core-database
// core-domain → core-common（純粋Kotlin、Android依存なし）
// core-network → core-common
// core-database → core-common

// ❌ feature → feature（feature間の直接依存禁止）
// ❌ core-domain → core-data（逆方向依存禁止）
```

---

## まとめ

| モジュール層 | 役割 | 依存先 |
|-------------|------|--------|
| app | DIグラフ・Navigation | feature, core |
| feature-* | 画面単位 | core-domain, core-ui |
| core-data | Repository | core-domain, network, db |
| core-domain | UseCase・Entity | core-common |
| core-ui | 共通UI | なし |

- Convention Pluginでビルド設定統一
- feature間は直接依存せずNavigationで接続
- core-domainは純粋Kotlin（テスト容易）
- 並列ビルドで高速化

---

8種類のAndroidアプリテンプレート（マルチモジュール設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
- [Version Catalog](https://zenn.dev/myougatheaxo/articles/android-compose-version-catalog-bom-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
