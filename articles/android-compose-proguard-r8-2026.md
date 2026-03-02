---
title: "ProGuard/R8完全ガイド — 難読化/最適化/トラブルシューティング"
emoji: "🔧"
type: "tech"
topics: ["android", "kotlin", "proguard", "security"]
published: true
---

## この記事で学べること

**ProGuard/R8**（難読化設定、keepルール、リフレクション対応、マッピングファイル、トラブルシューティング）を解説します。

---

## 基本設定

```kotlin
// build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true    // コード圧縮+難読化
            isShrinkResources = true  // 未使用リソース削除
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

---

## keepルール

```proguard
# proguard-rules.pro

# --- データクラス（Retrofit/Moshi で使用）---
-keep class com.example.app.data.model.** { *; }

# --- Retrofit インターフェース ---
-keep,allowobfuscation interface com.example.app.data.api.** { *; }

# --- Enum ---
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# --- Parcelable ---
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# --- Kotlin Serialization ---
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt

-keepclassmembers @kotlinx.serialization.Serializable class ** {
    *** Companion;
    *** INSTANCE;
    kotlinx.serialization.KSerializer serializer(...);
}

# --- Navigation の @Serializable ルート ---
-keep @kotlinx.serialization.Serializable class * { *; }
```

---

## Retrofit/Moshi 専用ルール

```proguard
# Retrofit
-dontwarn retrofit2.**
-keep class retrofit2.** { *; }
-keepattributes Signature
-keepattributes Exceptions

# Moshi
-keep class com.squareup.moshi.** { *; }
-keep @com.squareup.moshi.JsonQualifier interface *
-keepclassmembers @com.squareup.moshi.JsonClass class * {
    <init>(...);
    ** *;
}

# OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**
```

---

## マッピングファイル

```kotlin
// リリースビルド時に生成: app/build/outputs/mapping/release/mapping.txt
// → Firebase CrashlyticsやPlay Consoleにアップロード

// Crashlytics 自動アップロード
// build.gradle.kts
plugins {
    id("com.google.firebase.crashlytics")
}

android {
    buildTypes {
        release {
            // マッピングファイルをCrashlyticsに自動アップロード
            configure<CrashlyticsExtension> {
                mappingFileUploadEnabled = true
            }
        }
    }
}
```

---

## トラブルシューティング

```kotlin
// ❌ リフレクション使用クラスがクラッシュ
// java.lang.ClassNotFoundException: a.b.c
// → keepルール追加

// ❌ Retrofit API呼び出しで空レスポンス
// → データクラスのフィールド名が難読化されている
// → @SerializedName or -keep で対応

// デバッグ: R8の出力確認
// build.gradle.kts
android {
    buildTypes {
        release {
            // R8の設定ファイル出力
            proguardFiles("proguard-rules.pro")
        }
    }
}

// usage.txt: 削除されたコード
// seeds.txt: keepされたクラス
// mapping.txt: 難読化マッピング
```

---

## まとめ

| 機能 | 効果 |
|------|------|
| `isMinifyEnabled` | コード圧縮+難読化 |
| `isShrinkResources` | 未使用リソース削除 |
| `-keep` | 難読化対象外 |
| `mapping.txt` | スタックトレース復元 |

- リリースビルドでは必ずR8を有効化
- Retrofit/Moshiのデータクラスはkeep必須
- `mapping.txt`をCrashlyticsにアップロード
- テスト: リリースビルドで全機能動作確認

---

8種類のAndroidアプリテンプレート（R8設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [App Bundle](https://zenn.dev/myougatheaxo/articles/android-compose-app-bundle-delivery-2026)
- [セキュリティ](https://zenn.dev/myougatheaxo/articles/android-compose-certificate-pinning-2026)
- [Gradle Convention Plugin](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-convention-plugin-2026)
