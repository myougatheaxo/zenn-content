---
title: "CLAUDE.mdの書き方完全ガイド — Claude Codeの出力を劇的に改善する設定ファイル"
emoji: "📝"
type: "tech"
topics: ["ClaudeCode", "AI", "開発環境", "プロンプト", "自動化"]
published: true
---

## CLAUDE.mdとは

Claude Codeのプロジェクトルートに置く**設定ファイル**です。

プロジェクト固有のルール、コーディング規約、アーキテクチャ方針をClaude Codeに永続的に伝えることができます。毎回プロンプトに書く必要がなくなります。

---

## 基本構成

```markdown
# プロジェクト名

## 概要
このプロジェクトの目的と概要

## 技術スタック
- Kotlin + Jetpack Compose
- Room Database
- MVVM + Repository パターン

## コーディング規約
- data classはイミュータブル（valのみ）
- Coroutineはsuspend funで統一
- 権限は最小限（INTERNET不要）

## ファイル構成
- data/ — Entity, DAO, Database
- repository/ — Repository
- viewmodel/ — ViewModel
- ui/ — Composable Screen
```

---

## 効果的なパターン4選

### パターン1: 技術スタック指定

```markdown
## 技術スタック
- 言語: Kotlin（Javaは使わない）
- UI: Jetpack Compose（XMLレイアウトは使わない）
- DB: Room Database
- DI: なし（手動DI）
- テスト: JUnit5 + Compose Testing
```

**効果**: AIが古い技術（Java、XMLレイアウト）を生成するのを防ぐ。

### パターン2: コードスタイル強制

```markdown
## コードスタイル
- インデント: 4スペース
- 関数名: camelCase
- クラス名: PascalCase
- 定数: UPPER_SNAKE_CASE
- 1行の最大長: 100文字
- ワイルドカードimport禁止
```

**効果**: チーム全体で一貫したコードスタイルを維持。

### パターン3: セキュリティルール

```markdown
## セキュリティ
- INTERNET権限を追加しない
- 広告SDKを使わない
- ユーザーデータを外部に送信しない
- APIキーをハードコードしない
- ProGuard難読化を必ず設定する
```

**効果**: AIがセキュリティリスクのあるコードを生成するのを防止。

### パターン4: 禁止事項リスト

```markdown
## 禁止事項
- XMLレイアウトファイルの作成
- Java言語の使用
- var（ミュータブル変数）の乱用
- Any型の使用
- コメントの自動生成（コードで自明な場合）
- 不要なログ出力
```

**効果**: 「やってほしくないこと」を明示することで出力品質が上がる。

---

## 実例: ハビットトラッカーのCLAUDE.md

```markdown
# Habit Tracker

## 概要
日々の習慣を記録するAndroidアプリ。

## 技術スタック
- Kotlin + Jetpack Compose + Material3
- Room Database（ローカル保存のみ）
- MVVM + Repository
- Coroutine + Flow

## アーキテクチャ
data/Habit.kt → data/HabitDao.kt → repository/HabitRepository.kt
→ viewmodel/HabitViewModel.kt → ui/HabitScreen.kt

## ルール
- INTERNET権限を絶対に追加しない
- 全データは端末内に保存
- Material3のdynamic colorを使用
- テストは書かなくてよい（プロトタイプ段階）
```

このCLAUDE.mdを置いた状態で `claude "ハビットトラッカーを作って"` と実行すると、上記の設計方針に沿ったコードが47秒で生成されます。

---

## レイヤー別CLAUDE.md

Claude Codeは3つのレイヤーでCLAUDE.mdを読みます：

| レイヤー | 場所 | 用途 |
|---------|------|------|
| グローバル | `~/.claude/CLAUDE.md` | 全プロジェクト共通（コードスタイル等） |
| ワークスペース | `{workspace}/CLAUDE.md` | ワークスペース固有の設定 |
| プロジェクト | `{project}/CLAUDE.md` | プロジェクト固有のルール |

下位レイヤーが上位を上書きします。

---

## やりがちなミス

### ミス1: 指示が曖昧

```markdown
# ✗ 悪い例
きれいなコードを書いて

# ○ 良い例
- 1関数は30行以内
- ネストは3レベルまで
- early returnを使う
```

### ミス2: 情報が多すぎる

CLAUDE.mdが1000行を超えると、AIが重要な指示を見落とす可能性があります。**200行以内**を目安に。

### ミス3: 矛盾する指示

```markdown
# ✗ 矛盾
- テストを必ず書く
- 最小限のコードで完成させる（テスト不要）
```

---

## まとめ

- CLAUDE.mdはClaude Codeの出力品質を劇的に改善する
- 技術スタック、コードスタイル、禁止事項を明記する
- 200行以内で簡潔に
- 3レイヤー構成で柔軟に運用可能

---

:::message
Claude Code活用のプロンプトパターン集を[Zenn Books](https://zenn.dev/myougatheaxo/books/claude-code-prompt-patterns)で公開中（¥980）。CLAUDE.mdのテンプレートも収録。
:::

---

## 関連記事

- [Claude Code vs Cursor vs GitHub Copilot 比較](https://zenn.dev/myougatheaxo/articles/claude-code-vs-cursor-copilot)
- [2026年版 AIコーディングツール完全ガイド](https://zenn.dev/myougatheaxo/articles/ai-coding-tools-complete-2026)
- [AIが生成したコードをベテランエンジニアにレビューしてもらった](https://zenn.dev/myougatheaxo/articles/ai-code-quality-review)
- [AIアプリ開発で月3万円の副収入ロードマップ](https://zenn.dev/myougatheaxo/articles/ai-side-income-roadmap-2026)

:::message
8種類のAndroidアプリテンプレートを[Gumroad](https://myougatheax.gumroad.com)で公開中。
:::
