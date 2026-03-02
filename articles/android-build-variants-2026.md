---
title: "Build Variants完全ガイド — debug/release/staging環境を使い分ける"
emoji: "🔀"
type: "tech"
topics: ["android", "gradle", "kotlin", "build"]
published: true
---

## この記事で学べること

Build Variants（ビルドバリアント）を使えば、**開発・ステージング・本番環境を1つのプロジェクトで管理**できます。API URLやログ設定を環境ごとに切り替える方法を解説します。

---

## Build Typesの設定

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isDebuggable = true
            buildConfigField("String", "BASE_URL",
                "\"https://api-dev.example.com\"")
        }

        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            buildConfigField("String", "BASE_URL",
                "\"https://api.example.com\"")
        }

        create("staging") {
            initWith(getByName("debug"))
            applicationIdSuffix = ".staging"
            buildConfigField("String", "BASE_URL",
                "\"https://api-staging.example.com\"")
        }
    }
}
```

---

## buildConfigFieldの活用

```kotlin
// コードからアクセス
val baseUrl = BuildConfig.BASE_URL
val isDebug = BuildConfig.DEBUG

// Retrofitで使用
val retrofit = Retrofit.Builder()
    .baseUrl(BuildConfig.BASE_URL)
    .build()
```

---

## Product Flavors

```kotlin
android {
    flavorDimensions += "version"

    productFlavors {
        create("free") {
            dimension = "version"
            applicationIdSuffix = ".free"
            buildConfigField("Boolean", "IS_PREMIUM", "false")
            buildConfigField("Int", "MAX_ITEMS", "10")
        }

        create("premium") {
            dimension = "version"
            applicationIdSuffix = ".premium"
            buildConfigField("Boolean", "IS_PREMIUM", "true")
            buildConfigField("Int", "MAX_ITEMS", "999")
        }
    }
}
```

### バリアントの組み合わせ

```
freeDebug     ← 無料版 + デバッグ
freeRelease   ← 無料版 + リリース
premiumDebug  ← 有料版 + デバッグ
premiumRelease ← 有料版 + リリース
```

---

## 環境ごとのリソース

```
app/src/
├── main/           ← 共通
│   └── res/
├── debug/          ← デバッグ専用
│   └── res/
│       └── values/
│           └── strings.xml  ← app_name = "MyApp (Debug)"
├── staging/        ← ステージング専用
│   └── res/
└── release/        ← リリース専用
    └── res/
```

---

## アプリアイコンの切り替え

```
app/src/
├── debug/
│   └── res/
│       └── mipmap-xxxhdpi/
│           └── ic_launcher.webp  ← デバッグ用アイコン
├── staging/
│   └── res/
│       └── mipmap-xxxhdpi/
│           └── ic_launcher.webp  ← ステージング用アイコン
└── main/
    └── res/
        └── mipmap-xxxhdpi/
            └── ic_launcher.webp  ← 本番用アイコン
```

デバッグ版にはバナー付きアイコンを使えば、**どのビルドか一目瞭然**。

---

## コードでの条件分岐

```kotlin
// デバッグビルドのみログ出力
if (BuildConfig.DEBUG) {
    Log.d("API", "Response: $response")
}

// Flavorで機能制限
if (BuildConfig.IS_PREMIUM) {
    showPremiumFeature()
} else {
    showUpgradeDialog()
}
```

---

## まとめ

| 概念 | 用途 |
|------|------|
| Build Types | debug/release/staging環境 |
| Product Flavors | free/premium版 |
| buildConfigField | 環境変数の切り替え |
| ソースセット | 環境ごとのリソース・コード |
| applicationIdSuffix | 同一端末に複数バージョン共存 |

---

8種類のAndroidアプリテンプレート（Build Variant設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Gradleビルド高速化](https://zenn.dev/myougatheaxo/articles/android-gradle-tips-2026)
- [Version Catalog入門](https://zenn.dev/myougatheaxo/articles/android-version-catalog-2026)
- [ProGuard/R8完全ガイド](https://zenn.dev/myougatheaxo/articles/android-proguard-r8-2026)
