---
title: "Claude Codeでセキュリティチェックを開発フローに組み込む"
emoji: "🤖"
type: "tech"
topics:
  - claudecode
  - セキュリティ
  - 自動化
  - OWASP
  - 開発効率
published: true
---

## セキュリティをあとから入れるのは難しい

「セキュリティは後でやる」→リリース直前に脆弱性発見→修正コスト10倍。

Claude Codeを使えば、コードを書くたびにリアルタイムでセキュリティチェックができる。

---

## Hooksでセキュリティチェックを自動化

`.claude/settings.json` でファイル保存のたびにスキャンを走らせる:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python scripts/security-scan.py ${file}"
          }
        ]
      }
    ]
  }
}
```

---

## セキュリティスキャンスクリプト例

`scripts/security-scan.py`:

```python
import sys, re, json

DANGEROUS_PATTERNS = [
    (r'exec\s*\(', "コマンドインジェクション: exec()使用"),
    (r'eval\s*\(', "コードインジェクション: eval()使用"),
    (r'innerHTML\s*=', "XSS: innerHTML直接代入"),
    (r'SELECT.*\+.*WHERE', "SQLインジェクション疑い"),
    (r'password\s*=\s*["']', "ハードコードパスワード"),
    (r'secret\s*=\s*["']', "ハードコードシークレット"),
    (r'http://', "非TLS通信"),
]

file_path = sys.argv[1]
with open(file_path) as f:
    content = f.read()

issues = []
for line_num, line in enumerate(content.split('
'), 1):
    for pattern, message in DANGEROUS_PATTERNS:
        if re.search(pattern, line, re.IGNORECASE):
            issues.append(f"L{line_num}: {message}")

if issues:
    print("⚠️ セキュリティ問題検出:")
    for issue in issues:
        print(f"  {issue}")
    sys.exit(1)
```

---

## カスタムコマンドでOWASP Top 10チェック

`.claude/commands/security-audit.md`:

```markdown
# セキュリティ監査

$ARGUMENTS のOWASP Top 10観点でのセキュリティ監査:

## A01: アクセス制御の不備
- 認証なしで到達できるエンドポイントがないか
- ロールチェックが正しく実装されているか

## A02: 暗号化の失敗
- 平文でのパスワード保存・送信がないか
- 弱い暗号アルゴリズム (MD5, SHA1) を使っていないか

## A03: インジェクション
- SQLクエリのパラメータバインドが使われているか
- コマンド実行でユーザー入力が使われていないか

## A07: 認証の失敗
- セッショントークンの有効期限が設定されているか
- ブルートフォース対策があるか

各問題: ファイル名:行番号 + 修正提案を出力すること。
```

---

## 依存関係の脆弱性チェック

```bash
# npm依存関係の脆弱性チェック (Hooks経由)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo "$CLAUDE_TOOL_INPUT" | grep -q "npm install" && npm audit --audit-level=high || true'"
          }
        ]
      }
    ]
  }
}
```

---

## シークレット漏洩防止

`.claude/commands/check-secrets.md`:

```markdown
# シークレット漏洩チェック

コミット前に以下を確認:

1. git diff HEAD でステージング内容を確認
2. 以下のパターンがコードに含まれていないか検索:
   - APIキー (sk-*, AKIA*, etc.)
   - パスワード (password=, passwd=)
   - プライベートキー (-----BEGIN)
   - JWTシークレット
   - .env ファイルの直接コミット

問題があれば即座に報告して、git reset でステージングを外す手順を示す。
```

---

## CIパイプラインへの組み込み

```yaml
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        run: |
          npm audit --audit-level=moderate
          python scripts/security-scan.py src/
```

---

## まとめ

| タイミング | ツール | チェック内容 |
|-----------|--------|------------|
| コード保存時 | Hooks | パターンマッチング |
| PR作成時 | /security-audit | OWASP Top 10 |
| コミット前 | /check-secrets | シークレット漏洩 |
| マージ前 | CI | 依存関係脆弱性 |

セキュリティを後回しにしない仕組みを作るのが、Claude Codeの使い方として最も効果が大きい。


---

この記事の内容はnoteで公開しているClaude Code完全攻略ガイド（全7章）から抜粋したもの。MCPの構築方法、カスタムスキル設計、マルチエージェント構成まで網羅している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
