---
title: "APKサイズを半分にする5つのテクニック — AI生成アプリの最適化"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "optimization", "gradle"]
published: true
---

## この記事で学べること

AIで生成したアプリのAPKサイズが大きすぎる場合の最適化方法。5つのテクニックで**APKサイズを30〜50%削減**できます。

---

## 1. R8（コード圧縮）を有効化

```kotlin
// build.gradle.kts (app)
android {
    buildTypes {
        release {
            isMinifyEnabled = true  // R8有効化
            isShrinkResources = true  // リソース圧縮
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

R8は未使用のコードを自動削除し、残りのコードを最適化します。リリースビルドでは必ず有効に。

**効果**: コードサイズ20〜40%削減

---

## 2. 未使用リソースの削除

`isShrinkResources = true`で未使用の画像・レイアウト・文字列を自動削除。

手動で確認する場合：
- Android Studio → **Refactor** → **Remove Unused Resources**

---

## 3. 画像をWebPに変換

```
PNG (500KB) → WebP (150KB)  // 70%削減
```

Android Studioで変換：
1. `res/drawable`フォルダで画像を右クリック
2. **Convert to WebP** を選択
3. 品質75%で十分

ベクター画像が使える場合はSVG → `VectorDrawable`に変換すると更に軽量。

---

## 4. 不要な依存関係の削除

```kotlin
// build.gradle.kts
dependencies {
    // ❌ 使っていないのに残っている
    // implementation("com.google.android.material:material:1.12.0")

    // ✅ Compose使用時はCompose Material3のみでOK
    implementation("androidx.compose.material3:material3")
}
```

AIが生成した`build.gradle.kts`には、使われていない依存関係が含まれることがあります。`./gradlew dependencies`で確認。

---

## 5. AAB（Android App Bundle）で公開

```bash
# APK（全アーキテクチャ含む）
./gradlew assembleRelease  # → 15MB

# AAB（必要なアーキテクチャのみ配信）
./gradlew bundleRelease    # → 8MB（端末には5MBだけDL）
```

Google PlayはAABを推奨。ユーザーの端末に必要なリソースだけが配信されます。

---

## サイズ比較

| 最適化 | APKサイズ |
|--------|----------|
| 未最適化 | 15MB |
| R8 + リソース圧縮 | 10MB |
| + WebP変換 | 8MB |
| + 不要依存削除 | 7MB |
| AAB配信 | 5MB（端末DLサイズ） |

---

## APK Analyzerで確認

Android Studio → **Build** → **Analyze APK** で、何がサイズを占めているか可視化できます。

よくある原因：
- `lib/` フォルダ（ネイティブライブラリ）が巨大
- `res/` フォルダに大きい画像がある
- `classes.dex` が肥大化（未使用コード）

---

## まとめ

1. **R8有効化** — `isMinifyEnabled = true`
2. **リソース圧縮** — `isShrinkResources = true`
3. **WebP変換** — 画像サイズ70%削減
4. **不要依存削除** — build.gradleの棚卸し
5. **AABで公開** — 端末に必要分だけ配信

---

8種類のAndroidアプリテンプレート（最適化設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [Gradle設定入門](https://zenn.dev/myougatheaxo/articles/android-gradle-basics-2026)
- [MVVM設計の全体像](https://zenn.dev/myougatheaxo/articles/android-mvvm-architecture-2026)
