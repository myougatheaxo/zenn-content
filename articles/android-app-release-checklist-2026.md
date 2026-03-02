---
title: "アプリリリースチェックリスト — 公開前に確認すべき30項目"
emoji: "✅"
type: "tech"
topics: ["android", "kotlin", "googleplay", "release"]
published: true
---

## この記事で学べること

Androidアプリを**Google Playに公開する前**に確認すべきチェックリストを解説します。

---

## コード品質

```
□ ProGuard/R8の難読化が有効
□ デバッグログの削除（Log.d/Log.v）
□ BuildConfig.DEBUGによる分岐確認
□ ハードコードされたAPI URL/キーの除去
□ テスト用コードの除去
□ ANR（Application Not Responding）の対策
```

---

## セキュリティ

```
□ HTTPS通信のみ使用（android:usesCleartextTraffic="false"）
□ APIキーはBuildConfigまたは暗号化ストレージに保存
□ WebViewのJavaScript有効化が必要な箇所のみ
□ SQLインジェクション対策（Room使用）
□ 入力バリデーション実装
□ 証明書ピンニングの検討
```

---

## パフォーマンス

```kotlin
// メモリリークチェック
// - CoroutineScopeの適切なキャンセル
// - viewModelScope使用
// - DisposableEffectでのリソース解放

// 画像最適化
// - Coilでキャッシュ有効
// - 適切なサイズでのリクエスト
AsyncImage(
    model = ImageRequest.Builder(context)
        .data(url)
        .size(300) // 必要なサイズだけロード
        .crossfade(true)
        .build(),
    contentDescription = null
)

// LazyColumnの最適化
LazyColumn {
    items(items, key = { it.id }, contentType = { it.type }) { item ->
        ItemRow(item)
    }
}
```

---

## UI/UX

```
□ ダークモード対応
□ 横画面対応（または固定指定）
□ タブレットレイアウト対応
□ Edge-to-Edge対応（Android 15+）
□ 日本語/英語の表示確認
□ エラー画面・空状態の表示
□ ローディング表示
□ 48dp以上のタッチターゲット
□ contentDescriptionの設定（a11y）
```

---

## ストア掲載情報

```
□ アプリ名（30文字以内）
□ 短い説明（80文字以内）
□ 詳細な説明（4000文字以内）
□ スクリーンショット（最低2枚、推奨8枚）
  - スマホ: 1080x1920
  - タブレット: 1200x1920（7インチ/10インチ）
□ アイコン（512x512 PNG）
□ フィーチャーグラフィック（1024x500）
□ カテゴリ選択
□ コンテンツレーティング（質問票回答）
□ プライバシーポリシーURL
```

---

## ビルド設定

```kotlin
// build.gradle.kts
android {
    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 26           // 最低API確認
        targetSdk = 35        // 最新API対象
        versionCode = 1       // 整数で毎回増加
        versionName = "1.0.0" // ユーザー表示用
    }

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

    bundle {
        language { enableSplit = true }
        density { enableSplit = true }
        abi { enableSplit = true }
    }
}
```

---

## 署名

```
□ リリース用キーストアの作成
□ キーストアのバックアップ（紛失するとアップデート不可）
□ Google Play App Signingの有効化
□ アップロード鍵の設定
□ 署名済みAABのビルド確認
```

---

## テスト

```
□ 実機テスト（最低3機種）
□ Android 14+での動作確認
□ 初回起動テスト
□ ネットワーク切断時の動作
□ バックグラウンドからの復帰
□ 画面回転時の状態保持
□ メモリ不足時のプロセスキル復帰
□ Google Playの内部テスト配信
```

---

## 公開後

```
□ Firebase Crashlyticsの設定
□ Google Analyticsの設定
□ ANRレポートの監視
□ レビューへの返信体制
□ 次バージョンの計画
```

---

## まとめ

リリース前に必ず確認：
1. **セキュリティ**: HTTPS、キー管理、入力検証
2. **パフォーマンス**: メモリリーク、画像最適化、LazyColumn
3. **UI/UX**: ダークモード、エラー画面、a11y
4. **ストア**: スクリーンショット、説明文、プライバシーポリシー
5. **ビルド**: ProGuard有効、versionCode更新、署名
6. **テスト**: 実機、オフライン、回転、内部テスト

---

8種類のAndroidアプリテンプレート（本番リリース品質設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開ガイド](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [APK最適化ガイド](https://zenn.dev/myougatheaxo/articles/android-apk-optimization-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
