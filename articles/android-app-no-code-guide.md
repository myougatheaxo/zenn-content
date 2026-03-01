---
title: "コーディング経験ゼロからAndroidアプリを11分で動かす方法【2026年】"
emoji: "📱"
type: "tech"
topics: ["Android", "Kotlin", "JetpackCompose", "AI", "ノーコード"]
published: true
---

## はじめに

「Androidアプリを作りたいけど、プログラミングがわからない」

2026年、その壁はもう存在しません。AIがコードを書き、あなたは**設定値を書き換えるだけ**です。

この記事では、コーディング経験ゼロの状態から11分でAndroidアプリを実機で動かす手順を解説します。

## 必要なもの

- PC（Windows / Mac / Linux）
- Android Studio（無料）
- Androidスマートフォン
- USBケーブル

## 全体の流れ

```
テンプレート取得 → アプリ名変更 → カラー変更 → ビルド → インストール
   30秒         2分        90秒      5分       2分     = 11分
```

## Step 1: テンプレートを取得（30秒）

AIが生成したKotlinプロジェクトをダウンロードします。

プロジェクト構成はこうなっています：

```
app/
├── src/main/java/com/example/app/
│   ├── data/
│   │   ├── Habit.kt          ← データモデル
│   │   ├── HabitDao.kt       ← データベース操作
│   │   └── AppDatabase.kt    ← DB設定
│   ├── repository/
│   │   └── HabitRepository.kt ← ビジネスロジック
│   ├── viewmodel/
│   │   └── HabitViewModel.kt  ← UI状態管理
│   └── ui/
│       ├── HabitScreen.kt     ← 画面
│       └── theme/
│           └── Theme.kt       ← テーマ設定
└── src/main/res/
    └── values/
        └── strings.xml        ← アプリ名
```

意味がわからなくても大丈夫。触るのは2ファイルだけです。

## Step 2: アプリ名を変更（2分）

`app/src/main/res/values/strings.xml` を開きます。

```xml
<!-- 変更前 -->
<string name="app_name">HabitTracker</string>

<!-- 変更後 -->
<string name="app_name">毎日の習慣</string>
```

アプリ名を好きな名前に変えるだけです。

## Step 3: カラーテーマを変更（90秒）

`ui/theme/Theme.kt` の中にカラー定義があります。

```kotlin
// 変更前
val Purple80 = Color(0xFFD0BCFF)
val PurpleGrey80 = Color(0xFFCCC2DC)

// 変更後（例：ブルー系）
val Blue80 = Color(0xFF90CAF9)
val BlueGrey80 = Color(0xFFB0BEC5)
```

カラーコードは「[material color picker](https://m3.material.io/styles/color/overview)」で選べます。6桁の数字を書き換えるだけ。

## Step 4: Android Studioでビルド（5分）

1. Android Studioでプロジェクトフォルダを開く
2. Gradle同期が自動で始まる（初回は少し待つ）
3. 緑の再生ボタン ▶ をクリック

または、ターミナルで：

```bash
./gradlew assembleDebug
```

## Step 5: スマホにインストール（2分）

1. スマホの「開発者オプション」を有効化
   - 設定 → デバイス情報 → ビルド番号を7回タップ
2. 「USBデバッグ」をON
3. USBケーブルで接続
4. Android Studioで実行 → スマホに自動インストール

## 生成されるコードの品質

「AIが書いたコードって大丈夫？」という疑問に対して。

90年代からコードを書いてるベテランエンジニアに見せた結果：

> 「正しい」

具体的には：

- **エラーハンドリングがある** — ユーザーが間違った操作をしても落ちない
- **データ構造が適切** — Room Database（SQLite）で安全にデータ保存
- **関心の分離** — DAO / Repository / ViewModel / Screen が明確に分離
- **Material3対応** — 最新のGoogleデザインガイドラインに準拠

## Google Playに公開するには

11分でスマホに入れた後、Google Playに公開するには追加ステップが必要です：

1. Google Playデベロッパーアカウント登録（$25、一回限り）
2. アプリアイコン作成
3. ストア掲載情報（スクリーンショット、説明文）
4. `./gradlew assembleRelease` で署名付きAPK生成
5. Play Consoleからアップロード

## まとめ

- AIが生成するAndroidアプリは**プロ品質**
- 非エンジニアがやることは**2ファイルの値を書き換える**だけ
- 11分でスマホに入る
- Google Play公開も追加ステップで可能

---

:::message
8種類のAndroidアプリテンプレート（習慣トラッカー、割り勘計算、タイマー、家計簿、予算管理、タスク管理、単位変換、会議タイマー）を[Gumroad](https://myougatheax.gumroad.com)で公開中。
全てKotlin + Jetpack Compose + Material3。ビルド手順書付き。
:::
