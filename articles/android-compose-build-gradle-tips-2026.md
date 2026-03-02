---
title: "build.gradle.kts設定ガイド — Compose/Version Catalog/署名"
emoji: "⚙️"
type: "tech"
topics: ["android", "kotlin", "gradle", "buildsystem"]
published: true
---

## この記事で学べること

Android開発の**build.gradle.kts**設定（Compose、Version Catalog、署名、ProGuard）を解説します。

---

## Compose設定

```kotlin
// build.gradle.kts (app)
android {
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"
    }

    buildFeatures {
        compose = true
    }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2024.12.01")
    implementation(composeBom)
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    debugImplementation("androidx.compose.ui:ui-tooling")
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
}
```

---

## Version Catalog

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.0.21"
compose-bom = "2024.12.01"
hilt = "2.51.1"
room = "2.6.1"

[libraries]
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

[plugins]
android-application = { id = "com.android.application", version = "8.7.0" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version = "2.0.21-1.0.25" }
```

```kotlin
// build.gradle.kts で使用
dependencies {
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)
    implementation(libs.compose.material3)
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
}
```

---

## リリースビルド/署名

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file(project.property("KEYSTORE_FILE") as String)
            storePassword = project.property("KEYSTORE_PASSWORD") as String
            keyAlias = project.property("KEY_ALIAS") as String
            keyPassword = project.property("KEY_PASSWORD") as String
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

## ProGuardルール

```proguard
# proguard-rules.pro

# Retrofit
-keepattributes Signature
-keepattributes *Annotation*
-keep class retrofit2.** { *; }

# Gson
-keep class com.example.app.data.model.** { *; }

# Room
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *

# Compose
-dontwarn androidx.compose.**
```

---

## まとめ

- Compose BOMでバージョン統一管理
- `libs.versions.toml`でVersion Catalog
- KSPでRoom/Hiltのコード生成（kapt→KSP移行推奨）
- リリースビルドは`isMinifyEnabled = true`+ProGuard
- 署名情報は`gradle.properties`に外出し
- `compileSdk = 35`でAndroid 15対応

---

8種類のAndroidアプリテンプレート（ビルド設定最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [マルチモジュール](https://zenn.dev/myougatheaxo/articles/android-compose-multi-module-nav-2026)
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/android-google-play-release-2026)
- [Hilt依存性注入](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
