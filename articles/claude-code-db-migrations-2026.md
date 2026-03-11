---
title: "Claude Codeで安全なDBマイグレーション：本番障害を防ぐ設計パターン"
emoji: "🗄️"
type: "tech"
topics: ["claudecode", "database", "postgresql", "prisma", "マイグレーション"]
published: true
---

## はじめに

DBマイグレーションは間違えると本番障害になる。制約なしに「NOT NULLで列を追加して」と頼むと、既存データがあるテーブルでは実行時にエラーが起きる。

CLAUDE.mdで安全なパターンを強制する。

---

## CLAUDE.mdにマイグレーションルールを書く

```markdown
## DBマイグレーションルール（Prisma + PostgreSQL）

### コマンド（間違えないこと）
- 開発: `npx prisma migrate dev`
- 本番: `npx prisma migrate deploy`（migrate devを本番で実行禁止）
- スキーマ変更後: `npx prisma generate` を必ず実行

### 安全な変更（追加のみ）
- NULLABLEな列の追加: 安全
- DEFAULT値付き列の追加: 安全（既存行に値が入る）
- インデックス追加: 安全（大テーブルはCONCURRENTLY使用）
- 新テーブルの追加: 安全

### 危険な変更（計画必須）
- 列の削除: コードから参照を外してから（逆はダメ）
- 列名の変更: 新列追加→データコピー→参照切り替え→旧列削除の順
- 型変更: データのコピーが必要
- NOT NULL制約の追加: 既存データへの影響確認必須

### デプロイ順序（厳守）
1. 新列を追加するマイグレーション（まずDB変更）
2. 新列も旧列も読むコードをデプロイ
3. 旧列を削除するマイグレーション（最後にDB変更）
※ 逆順にすると本番でエラーが発生する
```

---

## マイグレーションの安全性チェックを依頼する

```
このマイグレーションを本番に適用する前に安全性をレビューしてください。

対象テーブル：users（200万件）
ダウンタイム許容：なし

マイグレーション：
[prisma/migrations/xxx/migration.sqlの内容をここに貼る]

確認してほしいこと：
1. テーブルロックが発生しますか？発生する場合はどのくらいの時間？
2. 既存データに影響しますか？
3. ロールバック方法は？
4. 複数のマイグレーションに分けるべきですか？
```

---

## 大テーブルへの列追加パターン

100万件以上のテーブルに列を追加する際の安全なパターンをClaude Codeに生成させる：

```
500万件あるusersテーブルに last_login_at タイムスタンプ列を追加したい。

テーブルロックなしで以下の手順を実行する方法を教えてください：
1. NULLABLEで列を追加
2. 既存データを1万件ずつバッチ補完
3. インデックスをCONCURRENTLYで追加（PostgreSQL）
4. データが全部埋まったらNOT NULLに変更

各ステップのSQLを生成してください。
```

---

## 列の安全な削除手順

```
phone_number列をusersテーブルから削除したい。
コードがまだこの列を参照している可能性がある。

安全な削除計画を作成してください：
1. コードベース全体でphone_numberの参照を洗い出す
2. 参照を削除する順序
3. Prismaスキーマから削除する前に確認すること
4. 最終的な削除マイグレーション
```

---

## 危険なマイグレーションを検出するCIチェック

```python
# .github/scripts/check_migration.py
import sys
import pathlib
import re

dangerous = [
    (r'DROP\s+(TABLE|COLUMN)\s+(?!IF EXISTS)', "IF EXISTSなしのDROP"),
    (r'ALTER.*NOT NULL(?!.*DEFAULT)', "DEFAULT値なしのNOT NULL追加"),
    (r'TRUNCATE', "マイグレーション内のTRUNCATE"),
    (r'DROP\s+DATABASE', "データベース削除"),
]

exit_code = 0
for f in pathlib.Path(sys.argv[1]).rglob("*.sql"):
    content = f.read_text(encoding="utf-8")
    for pattern, msg in dangerous:
        if re.search(pattern, content, re.IGNORECASE):
            print(f"[DANGER] {f.name}: {msg}")
            exit_code = 1

sys.exit(exit_code)
```

GitHub Actionsに追加：

```yaml
- name: マイグレーション安全チェック
  run: python .github/scripts/check_migration.py prisma/migrations/
```

---

## まとめ

Claude Codeで安全なDBマイグレーション：

1. **CLAUDE.md** に安全/危険なパターンを明示
2. **本番適用前** にレビューを依頼する
3. **大テーブル** はCONCURRENTLY + バッチ処理
4. **列削除** はデプロイ順序を厳守（コード先、DB後）
5. **CIチェック** で危険なパターンを自動検出

---

*DBマイグレーションのレビューはCode Review Packの `/code-review` スキルで自動化できます。**Code Review Pack（¥980）** はPromptWorksで購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
