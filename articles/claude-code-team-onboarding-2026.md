---
title: "チーム全員でClaude Codeを使う：オンボーディングと標準化"
emoji: "👥"
type: "tech"
topics: ["claudecode", "チーム開発", "onboarding", "開発効率化", "github"]
published: true
---

## はじめに

個人でClaude Codeを使うのは簡単だが、チーム全員で使い始めるには準備が必要だ。

CLAUDE.mdの書き方、Hooksの共有、カスタムスキルの配布方法を整理する。

---

## チーム共通のCLAUDE.md

プロジェクトルートの `CLAUDE.md` をGitで管理する。これがチーム全員に適用されるデフォルト設定になる。

```markdown
# プロジェクト: my-app

## 必読: このプロジェクトのルール
Claude Codeはこのファイルを参照しています。変更する場合はPRでレビューしてください。

## 技術スタック
- Node.js 20 + TypeScript 5.x
- Express 4 + Prisma ORM
- PostgreSQL 16

## テスト
npm test (Vitest)

## コーディングルール
- any型禁止
- console.log禁止（logger.ts使用）
- 新機能には単体テスト必須
- マジックナンバーは定数化

## 変更禁止
- src/legacy/ 以下（本番影響大）
- package.json の依存関係（要レビュー）

## よく使うファイル
- DB: src/lib/db.ts
- 認証: src/middleware/auth.ts
- 共通型: src/types/index.ts
```

---

## チームHooksの共有

`.claude/hooks/` ディレクトリをGitで管理する。

```
.claude/
├── settings.json      # Hooks設定（Git管理）
└── hooks/
    ├── ts_quality.py  # TypeScript品質チェック（Git管理）
    ├── guard_bash.py  # 危険なBashコマンドをブロック（Git管理）
    └── README.md      # Hooks説明（Git管理）
```

```markdown
# .claude/hooks/README.md

## チームHooks説明

### ts_quality.py
TypeScript/TSXファイルを書いた後に自動実行：
- Prettierフォーマット
- ESLint自動修正
- console.log検出（警告）

### guard_bash.py
以下のコマンドを実行前にブロック：
- rm -rf（ルートや重要ディレクトリへ）
- NODE_ENV=productionでのテスト実行
```

---

## カスタムスキルの配布

`.claude/skills/` ディレクトリをGitで管理すれば、チーム全員が同じスキルを使える。

```
.claude/
└── skills/
    ├── code-review.md    # /code-review スキル
    ├── test-gen.md       # /test-gen スキル
    └── refactor.md       # /refactor-suggest スキル
```

スキルファイルの例（`code-review.md`）：

```markdown
---
name: code-review
description: コードレビューを実行する
---

以下の観点でコードをレビューしてください：

1. CLAUDE.mdのコーディングルール準拠
2. セキュリティ（OWASP Top 10）
3. パフォーマンス（N+1問題、不要なループ）
4. テストカバレッジ
5. 型安全性

問題があれば優先度付きで報告してください（Critical / High / Medium / Low）。
```

---

## オンボーディングチェックリスト

新しいチームメンバー向けのClaude Code設定リスト。

```markdown
## Claude Code セットアップ手順

1. Claude Codeインストール
   `npm install -g @anthropic-ai/claude-code`

2. 認証
   `claude auth login`

3. プロジェクトのCLAUDE.mdを確認
   プロジェクトルートの CLAUDE.md を読む

4. Hooksの動作確認
   `.claude/settings.json` の設定を確認

5. カスタムスキルの確認
   `claude skills` でスキル一覧を確認

6. 試しに動かす
   `claude` を起動して「このプロジェクトの技術スタックを教えて」と聞く
   → CLAUDE.mdの内容が返ってくれば設定完了
```

---

## チームでCLAUDE.mdを更新するルール

CLAUDE.mdはチームの「Claude Codeへの指示書」。更新ルールを決めておく。

```markdown
## CLAUDE.md 更新ルール
1. 変更は必ずPR経由（直接mainへのpush禁止）
2. 変更理由をPR説明に書く
3. 変更前後でClaude Codeの動作を確認してからマージ
4. 月1回、内容が現状に合っているかチームでレビュー
```

---

## まとめ

チームでClaude Codeを標準化する方法：

1. **CLAUDE.md** をGitで管理（チーム共通の設定）
2. **Hooks** を `.claude/hooks/` でGit管理（品質チェックの統一）
3. **カスタムスキル** を `.claude/skills/` でGit管理（レビュー・テストの統一）
4. **オンボーディング手順書** で新メンバーが即日使えるように

---

*チーム向けスキルパック（/code-review, /refactor-suggest, /test-gen）は **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
