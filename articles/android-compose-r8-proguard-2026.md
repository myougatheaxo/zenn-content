---
title: "R8/ProGuard完全ガイド — 難読化/最適化/サイズ削減"
emoji: "🛡️"
type: "tech"
topics: ["android", "kotlin", "proguard", "optimization"]
published: true
---

## この記事で学べること

**R8/ProGuard**（コード難読化、リソース縮小、keepルール、デバッグ方法）を解説します。

---

## 基本設定

```kotlin
// build.gradle.kts
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

## ProGuardルール

```proguard
# proguard-rules.pro

# Kotlinシリアライゼーション
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keepclassmembers class kotlinx.serialization.json.** { *** Companion; }

# Retrofit
-keepattributes Signature, Exceptions
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}
-dontwarn javax.annotation.**

# Room
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *

# Hilt
-keep class dagger.hilt.** { *; }
-keep class javax.inject.** { *; }

# データクラス（API レスポンス）
-keep class com.example.myapp.data.model.** { *; }

# Enum
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
```

---

## よくあるクラッシュと対処

```proguard
# リフレクション使用クラスが削除される
-keep class com.example.myapp.data.model.UserResponse { *; }

# Gsonフィールド名変更
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# Nativeメソッド
-keepclasseswithmembernames class * {
    native <methods>;
}
```

---

## デバッグ・最適化

```proguard
# 行番号保持（スタックトレース用）
-keepattributes SourceFile,LineNumberTable

# ログ削除（Releaseビルド）
-assumenosideeffects class android.util.Log {
    public static int v(...);
    public static int d(...);
    public static int i(...);
}

# 最適化パス
-optimizationpasses 5
```

```xml
<!-- res/raw/keep.xml リソース保持 -->
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@layout/activity_*,@drawable/ic_*"
    tools:discard="@layout/unused_*" />
```

---

## まとめ

| 機能 | 設定 | 効果 |
|------|------|------|
| コード縮小 | `isMinifyEnabled = true` | 未使用コード削除 |
| リソース縮小 | `isShrinkResources = true` | 未使用リソース削除 |
| 難読化 | R8自動 | クラス名短縮 |
| 最適化 | R8自動 | バイトコード最適化 |

- `-keep`で必要クラスを保持
- `mapping.txt`でスタックトレース復元
- Retrofit/Room/Hiltのkeepルールは必須
- リリース前に必ず動作確認

---

8種類のAndroidアプリテンプレート（R8設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-baseline-profile-2026)
- [Version Catalog](https://zenn.dev/myougatheaxo/articles/android-compose-version-catalog-bom-2026)
- [GitHub Actions CI/CD](https://zenn.dev/myougatheaxo/articles/android-compose-github-actions-ci-2026)
