---
title: "Compose R8最適化完全ガイド — Full Mode/ルール設定/APKサイズ削減"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "r8"]
published: true
---

## この記事で学べること

**Compose R8最適化**（R8 Full Mode、コード縮小、APKサイズ削減、トラブルシューティング）を解説します。

---

## R8基本設定

```groovy
// build.gradle
android {
    buildTypes {
        release {
            isMinifyEnabled = true      // コード縮小+難読化
            isShrinkResources = true    // 未使用リソース削除
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}

// gradle.properties
// R8 Full Modeを有効化（より積極的な最適化）
android.enableR8.fullMode=true
```

---

## R8ルール（Compose向け）

```proguard
# proguard-rules.pro

# Compose自体は追加ルール不要（R8対応済み）

# --- Kotlin Serialization ---
-keep,includedescriptorclasses class com.example.app.**$$serializer { *; }
-keepclassmembers class com.example.app.data.model.** {
    *** Companion;
}

# --- Retrofit APIインターフェース ---
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}
-keep,allowobfuscation,allowshrinking interface retrofit2.Call
-keep,allowobfuscation,allowshrinking class kotlin.coroutines.Continuation

# --- Navigation引数 ---
-keep class com.example.app.navigation.** { *; }

# --- APKサイズ確認 ---
# ./gradlew assembleRelease
# Build > Analyze APK... で確認
```

---

## APKサイズ最適化

```groovy
// build.gradle
android {
    // 不要な言語リソース除外
    defaultConfig {
        resourceConfigurations += listOf("ja", "en")
    }

    // 不要なABI除外
    splits {
        abi {
            isEnable = true
            reset()
            include("arm64-v8a", "armeabi-v7a")
            isUniversalApk = false
        }
    }

    buildFeatures {
        buildConfig = true
        compose = true
    }

    // ViewBinding等の不要な機能をOFF
    // viewBinding = false // Composeのみなら不要
}

// Bundle形式で配信（APKより小さい）
// ./gradlew bundleRelease
```

---

## まとめ

| 設定 | 効果 |
|------|------|
| `isMinifyEnabled` | コード縮小 30-50% |
| `isShrinkResources` | リソース削除 10-20% |
| `R8 Full Mode` | より積極的な最適化 |
| `resourceConfigurations` | 言語リソース限定 |

- R8はProGuardの後継（Googleが開発）
- Composeは追加ProGuardルール不要
- Retrofit/Serialization/Navigationは追加ルール必要
- App Bundle形式で配信サイズを最小化

---

8種類のAndroidアプリテンプレート（R8最適化設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ProGuard](https://zenn.dev/myougatheaxo/articles/android-compose-compose-proguard-2026)
- [Compose BaselineProfile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
- [Compose AppStartup](https://zenn.dev/myougatheaxo/articles/android-compose-compose-app-startup-2026)
