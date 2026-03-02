---
title: "ProGuard/R8完全ガイド — 難読化・最適化でアプリを守る"
emoji: "🛡️"
type: "tech"
topics: ["android", "kotlin", "security", "proguard"]
published: true
---

## この記事で学べること

R8（ProGuardの後継）を使えば、コードの**難読化・最小化・最適化**を自動で行えます。リバースエンジニアリング対策とAPKサイズ削減を同時に実現します。

---

## R8を有効化

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true      // コード最小化
            isShrinkResources = true    // 未使用リソース削除
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

`isMinifyEnabled = true`でR8が有効になります。**リリースビルドでのみ有効化**するのが基本。

---

## proguard-rules.pro の基本

```proguard
# データクラスのシリアライズ対応（Gson/Moshi）
-keepclassmembers class com.example.myapp.data.model.** {
    <fields>;
}

# Retrofit のインターフェース
-keep interface com.example.myapp.data.api.** { *; }

# Room のエンティティ
-keep class com.example.myapp.data.entity.** { *; }

# Compose の @Composable 関数名を保持（デバッグ用）
-keepnames class * {
    @androidx.compose.runtime.Composable <methods>;
}
```

---

## よくあるクラッシュと対策

### 1. Gsonでシリアライズエラー

```proguard
# Gsonで使うデータクラスのフィールド名を保持
-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
```

### 2. Retrofit の呼び出し失敗

```proguard
# Retrofit
-keepattributes Signature
-keepattributes *Annotation*
-keep class retrofit2.** { *; }
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}
```

### 3. Roomのクエリエラー

```proguard
# Room
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *
-keepclassmembers class * {
    @androidx.room.* <methods>;
}
```

---

## R8のモード

| モード | 設定 | 効果 |
|--------|------|------|
| 互換モード | デフォルト | ProGuard互換 |
| フルモード | `android.enableR8.fullMode=true` | より積極的な最適化 |

```properties
# gradle.properties
android.enableR8.fullMode=true
```

フルモードはより多くの未使用コードを削除しますが、keep規則が不足していると**クラッシュする可能性**があります。

---

## デバッグ方法

### マッピングファイル

```
app/build/outputs/mapping/release/mapping.txt
```

難読化されたスタックトレースを元のコードに変換：

```bash
# retrace コマンド
retrace mapping.txt stacktrace.txt
```

### Play Console

Google Play Consoleにマッピングファイルをアップロードすれば、クラッシュレポートが**自動で元のコード名に変換**されます。

---

## 使い方の判断フロー

```
リリースビルド？
├── Yes → isMinifyEnabled = true
│         ├── Gson/Moshiを使用？ → データクラスのkeep追加
│         ├── Retrofitを使用？ → インターフェースのkeep追加
│         ├── Roomを使用？ → エンティティのkeep追加
│         └── リフレクション使用？ → 対象クラスのkeep追加
└── No → isMinifyEnabled = false（デバッグ時は無効）
```

---

## まとめ

- `isMinifyEnabled = true` + `isShrinkResources = true`でリリースビルド最適化
- データクラス・APIインターフェースは`-keep`で保護
- `mapping.txt`でスタックトレースを逆変換
- フルモードでさらに積極的な最適化
- リフレクションを使うライブラリは必ずkeep規則を追加

---

8種類のAndroidアプリテンプレート（R8設定・ProGuardルール構成済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [APKサイズ最適化](https://zenn.dev/myougatheaxo/articles/android-apk-optimization-2026)
- [アプリ署名完全ガイド](https://zenn.dev/myougatheaxo/articles/android-signing-release-2026)
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
