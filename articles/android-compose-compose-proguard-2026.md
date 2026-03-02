---
title: "Compose ProGuard完全ガイド — R8最適化/難読化/ルール設定/Room対応"
emoji: "🛡️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "proguard"]
published: true
---

## この記事で学べること

**Compose ProGuard**（R8最適化、難読化ルール、Room/Retrofit対応、リリースビルド設定）を解説します。

---

## 基本設定

```groovy
// build.gradle
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

---

## proguard-rules.pro

```proguard
# --- Kotlin Serialization ---
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt

-keepclassmembers class kotlinx.serialization.json.** { *** Companion; }
-keepclasseswithmembers class kotlinx.serialization.json.** {
    kotlinx.serialization.KSerializer serializer(...);
}

# --- Retrofit ---
-keepattributes Signature, Exceptions
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}
-dontwarn okhttp3.**
-dontwarn retrofit2.**

# --- Room ---
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *
-keepclassmembers class * { @androidx.room.* <methods>; }

# --- データクラス（APIレスポンス等） ---
-keep class com.example.app.data.model.** { *; }

# --- Compose ---
# Compose自体はR8対応済み、追加ルール不要

# --- Enum ---
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
```

---

## デバッグ用マッピング

```groovy
// build.gradle
android {
    buildTypes {
        release {
            // mapping.txtを生成（クラッシュレポート解析用）
            isMinifyEnabled = true

            // Firebase Crashlytics用
            firebaseCrashlytics {
                mappingFileUploadEnabled = true
            }
        }
    }
}
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `isMinifyEnabled` | コード縮小/難読化 |
| `isShrinkResources` | 未使用リソース削除 |
| `-keep` | 難読化除外 |
| `mapping.txt` | 難読化マッピング |

- `isMinifyEnabled = true`でR8コード最適化
- Retrofit/Room/Serializationは追加ルールが必要
- Composeは追加ProGuardルール不要
- `mapping.txt`をCrashlyticsにアップロードしてクラッシュ解析

---

8種類のAndroidアプリテンプレート（リリースビルド最適化対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose BaselineProfile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
- [Compose Benchmark](https://zenn.dev/myougatheaxo/articles/android-compose-compose-benchmark-2026)
- [Compose AppStartup](https://zenn.dev/myougatheaxo/articles/android-compose-compose-app-startup-2026)
