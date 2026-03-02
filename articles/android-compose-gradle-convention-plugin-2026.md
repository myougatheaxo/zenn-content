---
title: "Gradle Convention Plugin完全ガイド — ビルド設定の共通化/Version Catalog連携"
emoji: "🏗️"
type: "tech"
topics: ["android", "kotlin", "gradle", "jetpackcompose"]
published: true
---

## この記事で学べること

**Gradle Convention Plugin**（ビルドロジック共通化、Version Catalog連携、マルチモジュール設定、カスタムタスク）を解説します。

---

## Convention Pluginとは

マルチモジュールプロジェクトでビルド設定を共通化するプラグイン。`build-logic`モジュールに定義します。

```
project/
├── build-logic/
│   ├── convention/
│   │   ├── build.gradle.kts
│   │   └── src/main/kotlin/
│   │       ├── AndroidApplicationConventionPlugin.kt
│   │       ├── AndroidLibraryConventionPlugin.kt
│   │       └── AndroidComposeConventionPlugin.kt
│   └── settings.gradle.kts
├── app/
├── feature/
│   ├── home/
│   └── settings/
└── core/
    ├── data/
    └── ui/
```

---

## build-logic設定

```kotlin
// build-logic/settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
rootProject.name = "build-logic"
include(":convention")

// build-logic/convention/build.gradle.kts
plugins {
    `kotlin-dsl`
}
dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.compose.gradlePlugin)
}
```

---

## Android Library Plugin

```kotlin
// AndroidLibraryConventionPlugin.kt
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

            val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")
            dependencies {
                add("implementation", libs.findLibrary("androidx-core-ktx").get())
                add("testImplementation", libs.findLibrary("junit").get())
            }
        }
    }
}
```

---

## Compose Plugin

```kotlin
class AndroidComposeConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")

            extensions.configure<BaseExtension> {
                buildFeatures.compose = true
            }

            val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")
            dependencies {
                val bom = libs.findLibrary("compose-bom").get()
                add("implementation", platform(bom))
                add("implementation", libs.findLibrary("compose-ui").get())
                add("implementation", libs.findLibrary("compose-material3").get())
                add("implementation", libs.findLibrary("compose-ui-tooling-preview").get())
                add("debugImplementation", libs.findLibrary("compose-ui-tooling").get())
            }
        }
    }
}
```

---

## プラグイン登録

```kotlin
// build-logic/convention/build.gradle.kts
gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "myapp.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidLibrary") {
            id = "myapp.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidCompose") {
            id = "myapp.android.compose"
            implementationClass = "AndroidComposeConventionPlugin"
        }
    }
}
```

---

## featureモジュールでの使用

```kotlin
// feature/home/build.gradle.kts
plugins {
    id("myapp.android.library")
    id("myapp.android.compose")
}

dependencies {
    implementation(project(":core:ui"))
    implementation(project(":core:data"))
}
```

---

## まとめ

| 要素 | 役割 |
|------|------|
| Convention Plugin | ビルドロジック共通化 |
| Version Catalog | 依存関係バージョン管理 |
| `build-logic` | プラグイン定義モジュール |
| `gradlePlugin` | プラグインID登録 |

- `build-logic`でビルド設定を一元管理
- Convention Pluginで重複排除
- Version Catalogで依存関係バージョン統一
- Now in Android公式サンプルと同じ構成

---

8種類のAndroidアプリテンプレート（Convention Plugin採用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [マルチモジュール](https://zenn.dev/myougatheaxo/articles/android-compose-multi-module-2026)
- [Version Catalog](https://zenn.dev/myougatheaxo/articles/android-compose-version-catalog-bom-2026)
- [GitHub Actions CI](https://zenn.dev/myougatheaxo/articles/android-compose-github-actions-ci-2026)
