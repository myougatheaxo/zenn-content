---
title: "Claude Codeのメモリ・コンテキスト管理でAIの精度を維持する"
emoji: "🤖"
type: "tech"
topics:
  - claudecode
  - ai
  - 開発効率
  - コンテキスト管理
  - tips
published: true
---

## なぜコンテキスト管理が重要か

Claude Codeは会話が長くなると:
- トークンコストが増える
- 古い情報が精度を下げる
- `/clear` するとコンテキストがリセットされる

適切なメモリ設計で「必要な情報を必要な時だけ渡す」ができれば、コストを下げつつ精度を維持できる。

---

## 3層のコンテキスト管理

```
1. CLAUDE.md        ← 常時ロード (全会話で使用)
2. memory/ ファイル  ← 必要な時だけ読み込む
3. 会話コンテキスト  ← /clear でリセット可能
```

---

## CLAUDE.mdの設計原則

毎回ロードされるので、**絶対に必要な情報だけ**に絞る。

```markdown
# CLAUDE.md (50行以内が目標)

## 開発原則 (3行)
- 最小限の変更のみ
- テスト必須
- OWASP準拠

## 技術スタック
Node.js 20 / TypeScript 5 / Prisma / PostgreSQL

## 重要ファイル参照
- API設計: docs/api-design.md
- DB設計: docs/schema.md
- テスト方針: docs/testing.md
```

詳細は別ファイルに書いて「必要な時だけ読む」形にする。

---

## memory/ ディレクトリの活用

`.claude/memory/` に調査結果や決定事項を蓄積:

```
.claude/
  memory/
    api-endpoints.md       ← APIエンドポイント一覧
    db-schema-summary.md   ← スキーマ要約
    bug-patterns.md        ← よくあるバグパターン
    decisions.md           ← 設計判断の記録
```

例 - `decisions.md`:
```markdown
# 設計判断記録

## 2026-03-10: 認証方式
決定: JWT + refresh token (httpOnly cookie)
理由: XSS対策とUXのバランス
代替案: session-based (スケーリングが複雑なので却下)

## 2026-03-08: キャッシュ戦略
決定: Redis (TTL 5分)
対象: ユーザープロフィール、商品一覧
```

これで毎回「なぜこうなってるの？」という質問が不要になる。

---

## /clearを戦略的に使う

```bash
# タスクの区切りでクリア
/clear

# 直前の状態を引き継ぐには memory ファイルを更新しておく
```

**クリア前にやること:**
```
「今の作業状況をmemory/progress.mdにまとめて。
 次のセッションで引き継ぎに使うため」
```

---

## サブエージェントとコンテキスト分離

大量調査はサブエージェントに委譲:

```
メインコンテキスト (クリーンを維持)
  └─ サブエージェント (調査・整理)
       └─ 結果だけメインに返す
```

サブエージェントに「調査して memory/api-list.md に書いて」と指示すると、メインのコンテキストが汚れない。

---

## プロジェクト開始時のテンプレート

```bash
# 新プロジェクトの memory 初期化
mkdir -p .claude/memory
cat > .claude/memory/INIT.md << 'EOF'
# プロジェクト概要
(プロジェクトの目的・技術スタック)

# 現在の状況
(何が完成していて何が残っているか)

# 次のタスク
(具体的な次のアクション)
EOF
```

---

## まとめ

| 情報の種類 | 保存場所 | 読み込みタイミング |
|-----------|---------|-----------------|
| 常に必要な規約 | CLAUDE.md | 毎回自動 |
| 設計決定・パターン | memory/*.md | 必要時に明示指定 |
| 調査結果・一覧 | memory/*.md | 参照時のみ |
| 作業の一時メモ | (会話内) | /clearで消える |

コンテキストを意識的に設計することで、AIの「記憶喪失」問題を解決できる。


---

この記事の内容はnoteで公開しているClaude Code完全攻略ガイド（全7章）から抜粋したもの。MCPの構築方法、カスタムスキル設計、マルチエージェント構成まで網羅している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
