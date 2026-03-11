---
title: "Claude Codeでconsole.logを追放する：pinoで構造化ログを実装する"
emoji: "📋"
type: "tech"
topics: ["claudecode", "logging", "nodejs", "pino", "セキュリティ"]
published: true
---

## はじめに

Claude Codeが生成するコードに最も頻繁に混入するのが`console.log`だ。本番のAPIハンドラーに`console.log(user)`が残ると、ユーザーの個人情報がアプリケーションログに出力される可能性がある。

CLAUDE.mdでロガーを統一し、Hooksで自動的にブロックする設定を紹介する。

---

## CLAUDE.mdにロギングルールを書く

```markdown
## ロギングルール

### ロガー（必須）
- `console.log` / `console.error` を本番コードで使わない
- 代わりに: `src/lib/logger.ts` を使う（pino 構造化ログ）
- リクエストごとの子ロガー: `req.log`（requestId付き）

### ログレベルの使い方
- `error`: 5xxエラー・DB障害・サービス停止
- `warn`: リトライ・フォールバック・レート制限ヒット
- `info`: ユーザー登録・注文完了・ログイン成功
- `debug`: 開発環境専用（本番では出力しない）

### 記録すべきこと
- APIリクエスト入退（メソッド・URL・ステータス・所要時間）
- ビジネスイベント（user.created, payment.completed）
- 外部サービス呼び出し（開始・完了・失敗・所要時間）
- エラーの詳細（userId, requestId, エラーコード）

### 記録禁止（重要）
- パスワード・トークン・APIキー（平文・ハッシュ問わず）
- 個人情報（メール・電話番号・氏名・クレジットカード番号）
- リクエストボディ全文（機密情報が含まれる可能性）
```

---

## pinoロガーの生成

```
pinoを使った構造化ロガーモジュールを生成してください（CLAUDE.mdのルールに従う）。

要件：
- 本番環境(NODE_ENV=production): JSON出力
- 開発環境: pino-pretty でカラー表示
- デフォルトコンテキスト: { service: 'my-app', version: process.env.npm_package_version }
- 自動リダクト: password, token, authorization, cookie, *.password, *.token
- 子ロガー: logger.child({ userId, requestId }) でコンテキスト追加

保存先: src/lib/logger.ts
```

---

## リクエストロギングMWの生成

```
ExpressのHTTPリクエストロギングミドルウェアを生成してください。

要件：
- リクエスト開始時: method, url, requestId（ヘッダーになければ生成）
- リクエスト終了時: statusCode, 所要時間(ms)
- req.log = logger.child({ requestId }) をリクエストに添付
- リクエストボディはログしない（セキュリティ）
- /health, /ping エンドポイントは除外

保存先: src/middleware/requestLogger.ts
```

---

## console.logを自動ブロックするHook

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{"type": "command", "command": "python .claude/hooks/no_console_log.py"}]
      }
    ]
  }
}
```

```python
# .claude/hooks/no_console_log.py
import json, re, sys

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
fp = data.get("tool_input", {}).get("file_path", "")

# テストファイルとスクリプトは除外
if not fp or any(x in fp for x in ["test", "spec", "scripts", ".claude"]):
    sys.exit(0)

if not fp.endswith((".ts", ".js")):
    sys.exit(0)

if re.search(r'console\.log\(', content):
    if '// allow-console' not in content:
        print("[LOG] console.log禁止。src/lib/logger.ts を使用してください", file=sys.stderr)
        sys.exit(2)  # ブロック

sys.exit(0)
```

---

## PIIの自動マスキング

```
ログ出力前にPIIを自動マスキングするユーティリティを生成してください。

削除するフィールド（値を '***' に置換）:
- password, passwordHash, token, refreshToken, apiKey, secret

マスクするフィールド（部分表示）:
- email: user@example.com → user@***.com
- phone: 090-1234-5678 → ***-***-5678

仕様:
- 元のオブジェクトを変更しない（deepCopyを作る）
- ネストしたオブジェクトにも適用
- logger.tsのwriteメソッドで自動的に適用される

保存先: src/lib/logSanitizer.ts
```

---

## まとめ

Claude Codeでconsole.logを追放するための設定：

1. **CLAUDE.md** にロガーを統一、console.log禁止を明記
2. **pinoロガー** を生成させて構造化ログを実現
3. **Hooks** でconsole.logをファイル書き込み時に自動ブロック
4. **PIIマスキング** でロガーを通じた個人情報漏洩を防止

---

*console.logとPII漏洩リスクを自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
