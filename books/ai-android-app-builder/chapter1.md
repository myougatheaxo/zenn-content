---
title: "AIツール選び — Claude Code vs Cursor vs Copilot"
---

AIでAndroidアプリを作ろうとすると、最初の壁が「どのAIツールを使えばいいの？」問題です。

2024年〜2025年にかけて、コーディングAIが一気に増えました。Claude Code、Cursor、GitHub Copilot、Windsurf、Gemini Code Assist...選ぶだけで疲れます。

この章では、Androidアプリ開発に絞って3つの主要ツールを比較します。

---

## 3つのツールの立ち位置

| ツール | 形式 | 価格 | 向いている人 |
|--------|------|------|-------------|
| **Claude Code** | CLI（ターミナル） | $20/月〜 | コードの「意図」を理解させたい人 |
| **Cursor** | IDEエディタ | 無料〜$20/月 | VSCode感覚でAIを使いたい人 |
| **GitHub Copilot** | IDE拡張 | $10/月〜 | Android Studioを使い続けたい人 |

---

## Claude Code の特徴

Claude Codeは、Anthropicが開発したターミナルベースのAIエージェントです。

**強み：**

- 長い会話でもコンテキストを維持する能力が高い
- 「CLAUDE.md」でプロジェクトの規約を事前に設定できる
- ファイルを横断した変更が得意（複数ファイルを一度に修正）
- 「なぜこうするのか」を説明しながらコードを書いてくれる

**弱み：**

- ターミナル操作に慣れていないと最初の壁がある
- Android Studioとの直接連携はない（ファイルを手動でコピーする場面がある）

**Androidアプリ開発での使い方：**

```bash
# プロジェクトルートで起動
cd MyHabitApp
claude

# プロンプト例
> ViewModelを作って。HabitRepositoryから習慣一覧を取得して、
  UIStateとしてStateFlowで公開するやつ。
```

Claude Codeは「大きな設計を考える」のが得意です。アーキテクチャの相談、複数ファイルにまたがるリファクタリング、バグの根本原因調査など、複雑なタスクで真価を発揮します。

---

## Cursor の特徴

Cursorは、VSCodeをベースにしたAI統合IDEです。使い慣れたエディタ感覚でAIを活用できます。

**強み：**

- コードをリアルタイムで補完（タイプしながらAIが次の行を提案）
- `Cmd+K`（Mac）/ `Ctrl+K`（Windows）でインライン編集
- `Cmd+L`でチャット形式の質問
- コードベース全体を「インデックス化」して参照できる

**弱み：**

- AndroidプロジェクトはGradleが絡むためビルドはAndroid Studioで行う
- 無料プランはGPT-4oベースでClaudeほどコンテキスト維持が強くない

**Androidアプリ開発での使い方：**

Cursorでコードを書き、Android Studioでビルド・実行するという**ハイブリッド運用**が現実的です。

```
Cursor（コーディング）→ Android Studio（ビルド・デバッグ・エミュレータ）
```

KotlinファイルはどちらのエディタでもOKなので、この組み合わせは普通に動きます。

---

## GitHub Copilot の特徴

GitHub Copilotは、Android Studioのプラグインとして直接動作します。

**強み：**

- Android Studioを離れずにAI補完が使える
- XMLレイアウトファイルの補完も対応
- Gradleのdependencies追加なども提案してくれる

**弱み：**

- 「提案を受け入れるかどうか」の判断は自分でやる必要がある
- 長い設計相談には向いていない（チャットはあるが短い返答が多い）
- 月額$10〜$19（個人）

**Androidアプリ開発での使い方：**

`Tab`キーを押すだけでコードが補完されます。特に繰り返しパターン（RecyclerView Adapter、Room DAO、ViewModel）の定型コードを書く速度が上がります。

---

## みょうがのおすすめ構成

正直に言います。**「一番いいAIツール」は存在しません。** ツールの使い方次第です。

でも、この本を読んでいる人が「初めてAndroidアプリを作る」なら、みょうがはこの構成をおすすめします：

### パターンA: 完全無料でとりあえず試す

1. **Claude.ai（ブラウザ版、無料）** で設計とコード生成
2. **Android Studio** でビルド・実行

無料プランでも十分なコードを生成できます。制限に引っかかったら翌日また使えます。

### パターンB: 本格的にやる（月額あり）

1. **Claude Code（$20/月）** でメインのコーディング
2. **Android Studio** でビルド・デバッグ

Claude Codeは「CLAUDE.md」でプロジェクトのルールを覚えさせられるので、2個目・3個目のアプリを作るときに効率が爆上がりします。

---

## AIへの指示の出し方（共通テクニック）

どのツールを使うにしても、**指示の出し方**が大事です。

**悪い指示：**
```
Androidアプリを作って
```

**良い指示：**
```
Kotlinで書かれたAndroidアプリを作ってください。
- アーキテクチャ: MVVM
- UI: Jetpack Compose
- データ永続化: Room
- 機能: 習慣を毎日チェックできるハビットトラッカー
- 一覧画面と詳細画面の2画面構成
- 最小API: 26（Android 8.0）
```

「何を」「どんな技術で」「どんな機能で」を明確にするほど、AIの出力品質が上がります。

**段階的に指示する：**

一気に全部作らせるより、段階を分けた方が確実です：

```
ステップ1: プロジェクト構成とファイル一覧を提案して
ステップ2: データモデル（Habitエンティティ）を作って
ステップ3: RepositoryとDAOを作って
ステップ4: ViewModelを作って
ステップ5: Composeの画面を作って
```

---

## Android Studioのセットアップ

どのAIツールを使うにしても、Android Studioは必要です。まだインストールしていない場合：

1. https://developer.android.com/studio にアクセス
2. 「Download Android Studio」をクリック
3. インストーラーを実行（デフォルト設定でOK）
4. 初回起動時に「SDK Setup」が走る（10〜20分かかる）

インストール後、新規プロジェクトを作成するとき：

- **テンプレート**: Empty Activity を選択
- **言語**: Kotlin
- **Minimum SDK**: API 26（Android 8.0）を推奨
- **Build configuration language**: Kotlin DSL（build.gradle.kts）

これで準備完了です。次の章から実際にアプリを作ります。

---

## まとめ

- **Claude Code**: 設計・複雑タスク向き。CLIに慣れるとパワフル
- **Cursor**: VSCode感覚でAI補完。ハイブリッド運用が現実的
- **Copilot**: Android Studio直接統合。定型コードの補完が速い
- 初心者は**Claude.ai（無料ブラウザ版）+ Android Studio**の組み合わせから始めるのが最もハードルが低い
- どのツールでも「指示の質」が出力の質を決める
