---
title: "Claude CodeのSecurity Packで脆弱性診断を自動化する"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "security", "owasp", "devops", "自動化"]
published: true
---

## はじめに

コードレビューで「セキュリティ的にどうだろう？」と気になりつつ、OWASP Top 10を全部チェックする時間なんてない——そんな状況、あるあるですよね。

Claude CodeのSecurity Packを使えば、`/security-audit`、`/secret-scanner`、`/deps-check` の3つのスキルでそのフローを自動化できます。

---

## /security-audit：OWASP Top 10を自動チェック

プロジェクトルートで実行するだけ。

```bash
/security-audit src/
```

**サンプル出力:**

```
[CRITICAL] A03: Injection
  src/api/user.py:42 - SQLクエリに未サニタイズの入力値が直接結合されています
  修正案: プリペアドステートメントを使用してください

[HIGH] A07: Auth Failure
  src/middleware/auth.py:18 - JWTの署名検証をスキップするフラグが残存
  修正案: verify=False を削除し、本番では必ず検証を有効化

[MEDIUM] A05: Security Misconfiguration
  config/settings.py:5 - DEBUG=True が本番環境でも有効になる可能性
  修正案: 環境変数 DJANGO_DEBUG で制御し、デフォルトをFalseに
```

重大度ごとに分類してくれるので、対応優先度が一目でわかります。

---

## /secret-scanner：ハードコードされたAPIキーを検出

Gitにコミットする前にスキャンしておくと安心です。

```bash
/secret-scanner .
```

**検出例:**

```python
# NG: こういうコードを検出してくれる
AWS_SECRET_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
OPENAI_API_KEY = "sk-proj-abc123..."
DB_PASSWORD = "production_password_2024"
```

```
[FOUND] src/config.py:12
  Pattern: AWS Secret Access Key
  Value: wJalrXUtnFEMI/K7MD...KEY (masked)
  推奨: AWS Secrets Manager または環境変数に移行してください

[FOUND] .env.example:8
  Pattern: OpenAI API Key
  注意: .env.example がGit管理下にある場合、実際の値が含まれていないか確認を
```

`.gitignore` に漏れがあってもここで拾えます。

---

## /deps-check：依存関係の脆弱性スキャン

`requirements.txt` や `package.json` を渡すだけ。

```bash
/deps-check requirements.txt
```

**出力例:**

```
Scanning 48 packages...

[HIGH] CVE-2024-3094 - requests==2.28.0
  CVSS: 8.1 - リダイレクト時に認証ヘッダーが漏洩する可能性
  修正: requests>=2.32.0 にアップグレード

[MEDIUM] CVE-2023-45853 - Pillow==9.5.0
  CVSS: 5.3 - 特定のTIFF解析でバッファオーバーフロー
  修正: Pillow>=10.0.1 にアップグレード

Summary: 2 HIGH, 1 MEDIUM, 0 LOW
自動修正コマンド: pip install --upgrade requests Pillow
```

手動でNVDを検索する必要がなくなります。

---

## CI/CDへの組み込み

GitHub Actionsと組み合わせると、PRのたびに自動チェックが走ります。

```yaml
# .github/workflows/security.yml
name: Security Audit

on: [pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Security Pack
        run: |
          claude /secret-scanner .
          claude /deps-check requirements.txt
          claude /security-audit src/
```

---

## まとめ

| スキル | 用途 | 検出対象 |
|--------|------|----------|
| `/security-audit` | OWASP Top 10チェック | SQLi, XSS, 認証不備など |
| `/secret-scanner` | 秘匿情報検出 | APIキー, パスワード, トークン |
| `/deps-check` | 依存関係スキャン | CVE番号付きの脆弱性 |

3つ合わせて使うと、コードレビュー前のセキュリティ確認がほぼ自動化できます。

---

## Security Pack を試してみる

**Security Pack (¥1,480)** は PromptWorks で販売中です。
`/security-audit`、`/secret-scanner`、`/deps-check` の3スキルセット。

→ **[PromptWorks で購入する](https://prompt-works.jp)**

Claude Codeのガイド・テンプレート類はこちら:
→ **[Gumroad ストア](https://myougatheaxo.gumroad.com)**
