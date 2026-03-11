---
title: "Claude Codeにシークレットを漏洩させない：Hooksによる自動スキャン"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "security", "hooks", "apikey", "セキュリティ"]
published: true
---

## はじめに

Claude Codeがコードを生成する際、プロンプトに含まれた接続情報をそのままコードに埋め込む場合がある。

「postgres://admin:password@db.example.com/mydb に接続して」と伝えると、その文字列がそのままソースコードに含まれる可能性がある。

この問題を防ぐには2つのアプローチが有効だ。

---

## アプローチ1: CLAUDE.mdで「ハードコード禁止」を宣言する

```markdown
## セキュリティルール（必須）

### シークレット管理
- 認証情報・APIキー・パスワードをコードに直接書かない
- 全てのシークレットは環境変数から取得する
- パターン: `os.getenv("DATABASE_URL")` （ハードコード禁止）
- `.env` ファイルは読み書きしない（Claude Codeが触るべきではない）

### 禁止パターン
- `os.getenv("KEY", "実際のパスワード")` — デフォルト値にシークレット不可
- localhost以外のURLに認証情報を含めない
- base64エンコードしたシークレット（難読化ではない）
```

---

## アプローチ2: Hooksでファイル書き込み時に自動スキャン

ファイルが書き込まれた直後に自動でシークレットを検出するHookを設定する。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "python .claude/hooks/scan_secrets.py"
        }]
      }
    ]
  }
}
```

```python
# .claude/hooks/scan_secrets.py
import json
import re
import sys

PATTERNS = [
    (r'password\s*=\s*["\'][^"\']{4,}["\']', "ハードコードされたパスワード"),
    (r'secret\s*=\s*["\'][^"\']{8,}["\']', "ハードコードされたシークレット"),
    (r'api.?key\s*=\s*["\'][a-zA-Z0-9_\-]{16,}["\']', "ハードコードされたAPIキー"),
    (r'sk-[a-zA-Z0-9]{40,}', "OpenAI/Anthropic APIキーパターン"),
    (r'postgres://[^:]+:[^@]+@', "認証情報付きDB URL"),
    (r'mysql://[^:]+:[^@]+@', "認証情報付きMySQL URL"),
    (r'ghp_[a-zA-Z0-9]{36}', "GitHub Personal Access Token"),
]

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
file_path = data.get("tool_input", {}).get("file_path", "")

# テストファイルと.env.exampleはスキップ
if not content or any(x in file_path for x in ["test", ".env.example"]):
    sys.exit(0)

findings = []
for pattern, label in PATTERNS:
    if re.search(pattern, content, re.IGNORECASE):
        findings.append(f"[SCAN] {label}の疑いがあります")

if findings:
    for f in findings:
        print(f, file=sys.stderr)
    print("[SCAN] 環境変数を使用してください", file=sys.stderr)
    sys.exit(2)  # 書き込みをブロック

sys.exit(0)
```

`sys.exit(2)` で書き込み操作をブロックする。Claude Codeはシークレットを除去してから再試行する。

---

## /secret-scannerスキルで既存コードをスキャン

Security Packに含まれるスキルで、既存コードを包括的にスキャンできる。

```bash
/secret-scanner src/
```

出力例：

```
シークレットスキャン結果
========================
高リスク:
  src/config/old_config.py:15 — ハードコードされたDBパスワード
  src/utils/email.py:8 — SMTPパスワードがハードコード

注意:
  src/tests/fixtures.py:23 — テスト用認証情報（テストのみなら問題なし）

クリーン:
  src/api/ — 問題なし
  src/services/ — 問題なし
```

---

## .env.exampleの維持

```
# .env.example（これをコミット、.envはコミットしない）
DATABASE_URL=postgres://user:password@localhost:5432/mydb
REDIS_URL=redis://localhost:6379
ANTHROPIC_API_KEY=sk-ant-...
STRIPE_SECRET_KEY=sk_test_...
JWT_SECRET=your-256-bit-secret-here
```

CLAUDE.mdに追加：

```markdown
## 環境変数ドキュメント
- 新しい環境変数は必ず .env.example にプレースホルダー付きで追加
- .env は絶対にコミットしない（.gitignoreに含まれている）
- 必須環境変数は起動時に `src/config/env.ts` でバリデーション
```

---

## 起動時バリデーションパターン

```typescript
// src/config/env.ts
function requireEnv(key: string): string {
  const value = process.env[key];
  if (!value) {
    throw new Error(`必須の環境変数 ${key} が設定されていません`);
  }
  return value;
}

export const config = {
  database: { url: requireEnv('DATABASE_URL') },
  auth: { jwtSecret: requireEnv('JWT_SECRET') },
  stripe: { secretKey: requireEnv('STRIPE_SECRET_KEY') },
};
```

CLAUDE.mdにこのパターンを書いておくと、Claude Codeが環境変数アクセスを統一してくれる。

---

## まとめ

Claude Codeにシークレットを漏洩させない2本柱：

1. **CLAUDE.md** に「ハードコード禁止」を明記する
2. **Hooks** でファイル書き込み時に自動スキャンする

どちらか一方ではなく、両方設定することで二重の防御になる。

---

*`/secret-scanner` スキルは **Security Pack（¥1,480）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
