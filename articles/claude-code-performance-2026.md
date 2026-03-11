---
title: "Claude Codeでパフォーマンス問題を特定・修正する"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "performance", "nodejs", "database", "最適化"]
published: true
---

## はじめに

「APIが遅い」という状況でClaude Codeをどう使うか。

「最適化して」と渡すだけでは、関係ない箇所まで変更されるリスクがある。パフォーマンス問題には診断→特定→修正のステップが必要だ。

---

## ステップ1: パフォーマンス問題を診断させる

```
このコードのパフォーマンス問題を診断してください。
特に以下の観点で：
- N+1問題
- 不要なデータ取得（SELECT *）
- メモリリーク
- 非効率なループ・再帰
- キャッシュできるのにしていない処理

修正はまだしないでください。

[コードをここに貼る]
```

---

## ステップ2: 計測ポイントを聞く

```
このAPIが遅いです。どこで時間がかかっているか計測するためのログを追加してください。

現在の実行時間：2.3秒（目標：500ms以下）
処理内容：ユーザーIDからダッシュボードデータを取得

[コードをここに貼る]
```

---

## よくあるN+1問題の修正

```
以下のコードにN+1問題があります。Prismaを使ってeager loadingで修正してください：

// 現在のコード（N+1）
const users = await prisma.user.findMany();
const usersWithPosts = await Promise.all(
  users.map(u => prisma.user.findUnique({
    where: { id: u.id },
    include: { posts: true }
  }))
);
```

Claude Codeが生成する修正：

```typescript
// 修正後（単一クエリ）
const users = await prisma.user.findMany({
  include: { posts: true }
});
```

---

## データベースクエリの最適化

```
以下のクエリが遅いです（実行時間：1.5秒）。

SQLクエリ：
[クエリをここに貼る]

テーブルサイズ：users(10万件), posts(50万件)
既存インデックス：users.email, posts.user_id

最適化の提案とインデックス追加の提案をしてください。
```

---

## CLAUDE.mdにパフォーマンスルールを書く

```markdown
## パフォーマンスルール
- SELECT * は禁止（必要なカラムを明示）
- N+1問題を避ける（include/joinを使う）
- ページネーションなしで全件取得しない（最大1000件）
- 外部APIの呼び出しは並列化する（Promise.all）
- キャッシュ対象: ユーザーマスタ・設定値（TTL: 5分）
- ファイル操作・DB操作の結果は変数に入れてループ外に出す
```

---

## パフォーマンス問題を防ぐHook

```python
# .claude/hooks/check_performance.py
import json, re, sys

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
fp = data.get("tool_input", {}).get("file_path", "")

if not content or not fp.endswith((".ts", ".js", ".py")):
    sys.exit(0)

warnings = []

# SELECT * チェック
if re.search(r'SELECT\s+\*', content, re.IGNORECASE):
    warnings.append("SELECT * が検出されました。必要なカラムを明示してください")

# ループ内クエリの疑いパターン（簡易チェック）
if re.search(r'for.{0,50}await.{0,50}(find|query|get)', content, re.DOTALL):
    warnings.append("ループ内でのDB操作の疑いがあります（N+1問題）")

if warnings:
    for w in warnings:
        print(f"[PERF] {w}", file=sys.stderr)

sys.exit(0)
```

---

## まとめ

Claude Codeでのパフォーマンス最適化：

1. **診断先行**（問題のある箇所を特定してから修正）
2. **計測ポイント**を追加してボトルネックを確認
3. **N+1問題**はeager loadingで修正
4. **CLAUDE.md** にパフォーマンスルールを明示
5. **Hooks** でSELECT *やループ内クエリを検出

---

*パフォーマンス問題を自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
