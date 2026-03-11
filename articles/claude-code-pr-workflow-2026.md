---
title: "Claude CodeでPRワークフロー全体を自動化する"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "github", "githubactions", "codereview", "開発効率化"]
published: true
---

## はじめに

PR（プルリクエスト）のワークフローは、コードを書く以外にも多くの作業がある。

- PR説明文を書く
- 自己レビューをする
- レビュアーのコメントに対応する
- リリースノートを書く

これらのほとんどをClaude Codeで自動化できる。

---

## PR説明文の生成

変更が完了したら：

```bash
git diff main...HEAD | claude -p "この変更のPR説明文を書いてください。

含めるもの：
- 概要（2-3文）
- 変更内容と理由
- テスト内容
- スクリーンショットが必要か（UIの変更の場合）
- 破壊的変更の有無

GitHub Markdownで書いてください。"
```

または：

```
以下の変更についてPR説明文を書いてください。

変更ファイル：
- src/services/user.service.ts — メール認証機能を追加
- src/routes/auth.ts — /verify-emailエンドポイントを追加
- tests/services/user.service.test.ts — 認証テストを追加

目的：ユーザーが保護されたルートにアクセスする前にメール認証を必須にする
```

---

## PR前の自己レビュー

PRを出す前に自分でレビューする：

```bash
git diff main...HEAD | claude -p "このコードをシニアエンジニアとしてレビューしてください。

確認ポイント：
1. 正確性（バグ、エッジケース）
2. セキュリティ（インジェクション、認証、シークレット漏洩）
3. パフォーマンス（N+1問題、不要なクエリ）
4. テスト不足
5. CLAUDE.mdのルール違反

問題があれば file:行番号 の形式で指摘してください。"
```

---

## GitHub Actions：PR時に自動レビュー

`.github/workflows/claude-review.yml`：

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: PR差分を取得
        run: git diff origin/main...HEAD > /tmp/pr.diff

      - name: Claude Codeレビュー実行
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          pip install anthropic
          python .github/scripts/claude_review.py /tmp/pr.diff

      - name: PRにコメント投稿
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('/tmp/review.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: review
            });
```

---

## マージコンフリクトの解消

```
マージコンフリクトが発生しています。

<<<<<<< HEAD
[自分のバージョン]
=======
[相手のバージョン]
>>>>>>> feature/other-branch

背景：両ブランチともユーザープロフィール機能を追加しましたが、
HEADのバージョンはUserProfileクラスを使い、
相手のブランチはプレーンオブジェクトを使っています。
クラスを使うアプローチを採用したいです。
```

「なぜ両方のバージョンが存在するか」のコンテキストを渡すと、解消の精度が上がる。

---

## レビューコメントへの対応

レビュアーのコメントに：

```
レビュアーから以下のコメントをもらいました：
「このクエリはユーザーリストが増えるとN+1問題を引き起こします。
joinかeager loadingを検討してください。」

指摘されたコード：
[コードをここに貼る]

どう修正すればいいですか？修正版を書いてください。
```

---

## リリースノート生成

```
バージョン2.3.0のリリースノートを生成してください。

2.2.0以降にマージされたPR：
[PRのタイトルと説明のリストをここに貼る]

形式：ユーザー向けのchangelog。
内部リファクタリング・テストは含めない。
グループ：機能追加 / バグ修正 / 破壊的変更
```

---

## CLAUDE.mdにPRルールを書く

```markdown
## PRワークフロー

### PR前のチェックリスト
1. /code-review で変更ファイルをレビュー
2. テストカバレッジが落ちていないか確認
3. ユーザー向け変更なら CHANGELOG.md を更新
4. 新しい環境変数があれば .env.example を更新

### PR説明のテンプレート
- 問題: このPRが解決する課題
- 解決策: 採用したアプローチ
- テスト: 動作確認の方法
- 破壊的変更: なし / 影響の説明
```

---

## まとめ

Claude CodeでのPRワークフロー自動化：

1. **PR説明文**: `git diff | claude -p` で自動生成
2. **自己レビュー**: PRを出す前にシニアエンジニア目線でチェック
3. **CI統合**: GitHub ActionsでPR時に自動レビュー
4. **コンフリクト解消**: コンテキストを渡して解消方針を決める

---

*`/code-review` スキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
