---
title: "Claude Codeカスタムスラッシュコマンドで繰り返し作業をゼロにする"
emoji: "🤖"
type: "tech"
topics:
  - claudecode
  - ai
  - 自動化
  - 開発効率
  - カスタマイズ
published: true
---

## カスタムスラッシュコマンドとは

Claude Codeでは `/commit` や `/review-pr` のような独自コマンドを作れる。

チームやプロジェクトで繰り返す作業を1コマンドにまとめることで、毎回同じ指示を書かなくて済む。

---

## 仕組み

`.claude/commands/` ディレクトリにMarkdownファイルを置くだけ。

```
.claude/
  commands/
    review.md       → /review
    deploy-check.md → /deploy-check
    db-migrate.md   → /db-migrate
```

---

## 実例1: コードレビューコマンド

`.claude/commands/review.md`:

```markdown
# コードレビュー

現在の変更内容 (`git diff HEAD`) をレビューして:

1. **バグリスク**: クラッシュやデータ破損の可能性
2. **セキュリティ**: OWASP Top 10の観点
3. **パフォーマンス**: N+1クエリ、不要なループ
4. **可読性**: 変数名、関数の長さ、複雑度
5. **テスト**: カバレッジ不足の箇所

各問題にはファイル名:行番号を明記すること。
重大度: 🔴必須修正 / 🟡推奨 / 🟢提案
```

使い方:
```bash
claude /review
```

---

## 実例2: デプロイ前チェックコマンド

`.claude/commands/deploy-check.md`:

```markdown
# デプロイ前チェック

本番デプロイ前に以下を確認:

## 自動実行
- [ ] `npm test` — 全テスト通過確認
- [ ] `npm run lint` — Lintエラーゼロ確認
- [ ] `npm run build` — ビルド成功確認

## 手動確認
- [ ] 環境変数 `.env.production` に必要な変数が設定されているか
- [ ] DBマイグレーションが実行済みか
- [ ] ロールバック手順が準備されているか

問題があれば止めて報告すること。全てOKなら「デプロイ準備完了」と出力。
```

---

## 実例3: DBマイグレーション生成コマンド

`.claude/commands/db-migrate.md`:

```markdown
# DBマイグレーション生成

$ARGUMENTS のスキーマ変更に対するマイグレーションファイルを生成:

1. `migrations/` 内の既存マイグレーションを確認
2. タイムスタンプ付きファイル名で新規マイグレーションを作成
3. UP (適用) と DOWN (ロールバック) の両方を書く
4. 既存データへの影響を明記
5. マイグレーション実行コマンドを出力

使用ORM: Prisma (prisma/schema.prisma を参照)
```

使い方:
```bash
claude /db-migrate "usersテーブルにlast_login_atカラムを追加"
```

---

## 実例4: PR作成コマンド

`.claude/commands/pr.md`:

```markdown
# PR作成

現在のブランチの変更からPRを作成:

1. `git log main..HEAD` で変更コミットを確認
2. `git diff main...HEAD` で差分を確認
3. 以下の形式でPR本文を作成:

```
## 概要
(変更の目的を1-3行で)

## 変更内容
- (変更点のリスト)

## テスト方法
- [ ] (確認手順)

## スクリーンショット (UIの場合)
```

4. `gh pr create` で実際にPRを作成
```

---

## チーム共有の設計ポイント

```
.claude/
  commands/          ← チームで共有 (git管理)
    review.md
    deploy-check.md

~/.claude/
  commands/          ← 個人設定 (git管理外)
    my-debug.md
    quick-fix.md
```

チーム共有コマンドは `.claude/commands/` に置いてgit管理。個人の作業効率化は `~/.claude/` に置く。

---

## 引数の使い方

コマンドに `$ARGUMENTS` を使うと実行時に値を渡せる:

```markdown
# テスト実行

`$ARGUMENTS` モジュールのテストを実行して、失敗した場合は原因を分析すること。
```

```bash
claude /test UserService
claude /test PaymentController
```

---

## まとめ

| コマンド例 | 効果 |
|-----------|------|
| /review | PR前の品質チェックを1コマンドに |
| /deploy-check | デプロイ前確認の標準化 |
| /db-migrate | マイグレーション生成の自動化 |
| /pr | PR作成フローの統一 |

繰り返し使う指示はすべてコマンド化する習慣をつけると、チーム全体の生産性が上がる。


---

この記事の内容はnoteで公開しているClaude Code完全攻略ガイド（全7章）から抜粋したもの。MCPの構築方法、カスタムスキル設計、マルチエージェント構成まで網羅している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
