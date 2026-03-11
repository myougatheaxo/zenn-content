---
title: "Claude Codeでシークレット漏洩を自動検出する - /secret-scanner スキルの実践"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "セキュリティ", "gitops", "devops", "APIキー"]
published: true
---

## はじめに

「GitHubにAWSのシークレットキーをpushしてしまった」——そんな事故のニュースを見るたびに、他人事とは思えないエンジニアも多いはずです。

実際、GitHubの公式レポートによれば、毎年数百万件のシークレットが誤ってリポジトリに公開されています。原因のほとんどは「うっかりハードコード」です。`.env` ファイルを `.gitignore` に追加し忘れた、デバッグ中に一時的に書いたキーをそのままコミットした、チームメンバーが設定ファイルを直接編集した——どれも起こりうるミスです。

Claude Codeの `/secret-scanner` スキルは、こうした漏洩をコミット前に自動で検出します。この記事では、スキルの仕組みから実際の出力、git hooksとの連携まで解説します。

---

## /secret-scanner の仕組み

`/secret-scanner` はコードベース全体を正規表現パターンでスキャンし、シークレットの疑いがある文字列を検出します。

### 検出対象のパターン

主要なプロバイダのキー形式をカバーしています。

| プロバイダ | パターン例 | 検出基準 |
|-----------|-----------|---------|
| AWS | `AKIA[0-9A-Z]{16}` | アクセスキーID形式 |
| AWS Secret | `[0-9a-zA-Z/+]{40}` | 40文字のBase64文字列 |
| GitHub | `ghp_[0-9a-zA-Z]{36}` | Personal Access Token |
| GitHub Actions | `github_pat_[0-9a-zA-Z_]{82}` | Fine-grained PAT |
| Anthropic | `sk-ant-[0-9a-zA-Z\-]{90,}` | Claude API Key |
| Stripe | `sk_live_[0-9a-zA-Z]{24,}` | 本番シークレットキー |
| JWT | `eyJ[a-zA-Z0-9\-_]+\.[a-zA-Z0-9\-_]+\.[a-zA-Z0-9\-_]+` | JSON Web Token |
| 汎用パスワード | `password\s*=\s*["'][^"']{8,}["']` | ハードコードされた文字列 |

スキャン対象はソースコード、設定ファイル、`.env` 系ファイル、Dockerfileなど、テキストとして読めるファイル全般です。

---

## 偽陽性フィルタリングの設計

正規表現だけでスキャンすると、テストコードやサンプル値まで検出してしまいます。`/secret-scanner` は以下のフィルタリングロジックで偽陽性を抑制しています。

### 1. テストファイルの除外

`test_*.py`、`*_test.go`、`*.spec.ts`、`__tests__/` 配下などテスト用パスは、デフォルトでは警告レベルを下げて表示します（無視はしない——テストコードに実キーが混入する事故も存在するため）。

### 2. プレースホルダーの除外

以下のような明らかなダミー値はスキップします。

```python
PLACEHOLDER_PATTERNS = [
    r"your[_\-]?(api[_\-]?)?key[_\-]?here",
    r"<YOUR[_\-]?.*?KEY>",
    r"xxxx+",
    r"1234567890",
    r"example\.com",
    r"dummy|placeholder|sample|test|fake",
]
```

### 3. エントロピー分析

高エントロピーな文字列はシークレットである可能性が高い、という原則を活用します。シャノンエントロピーが閾値（通常3.5以上）を超える文字列は優先的に報告されます。

```
entropy("password123")    → 2.75  (低エントロピー→可能性低い)
entropy("wJalrXUtnFEMI")  → 3.91  (高エントロピー→要注意)
```

---

## 実際の出力例

プロジェクトルートで実行します。

```bash
/secret-scanner .
```

**出力サンプル:**

```
/secret-scanner: Scanning 147 files...

[CRITICAL] src/config/aws.py:12
  Pattern   : AWS Secret Access Key
  Match     : wJalrXUtnFEMI/K7MD...KEY (masked)
  Entropy   : 4.21
  ⚠ WARNING : このキーがGit履歴に含まれている可能性があります
              コミット済みの場合は即座にAWSコンソールでキーを無効化してください
  推奨対応  : AWS Secrets Manager または環境変数 AWS_SECRET_ACCESS_KEY に移行

[CRITICAL] .env.backup:8
  Pattern   : Stripe Live Secret Key
  Match     : sk_live_4eC39H...qhvnJ (masked)
  Entropy   : 3.87
  ⚠ WARNING : .env.backup がGit管理下に入っていないか確認してください
  推奨対応  : .gitignore に *.backup を追加し、Stripeダッシュボードでキーをロールオーバー

[HIGH] src/auth/jwt_helper.py:34
  Pattern   : JWT Token (hardcoded)
  Match     : eyJhbGciOiJIUzI1NiIs... (masked)
  Note      : JWTの署名鍵がソースコードに含まれています
  推奨対応  : 環境変数 JWT_SECRET に移行し、定期的にロールオーバーする運用を検討

[INFO] tests/test_auth.py:19
  Pattern   : Generic API Key (test file)
  Match     : test_api_key_abc123
  Note      : テストファイル内の値です。実際のキーでないことを確認してください

---
Summary:
  CRITICAL : 2
  HIGH     : 1
  INFO     : 1
  Scanned  : 147 files in 0.8s
```

CRITICALが出た場合、スキャナはGit履歴への混入も警告します。コードから削除しただけでは不十分で、`git filter-repo` などで履歴からも除去する必要があります。

---

## git hooks連携

コミット前に自動でスキャンを走らせるには、pre-commitフックとして設定するのが最も効果的です。

### 手動設定（シンプルな方法）

`.git/hooks/pre-commit` に以下を追加します。

```bash
#!/bin/bash
# pre-commit: シークレット漏洩チェック
echo "Running /secret-scanner..."

# ステージングされたファイルのみスキャン
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)

if [ -z "$STAGED_FILES" ]; then
  exit 0
fi

# Claude Code CLIでスキャン
claude /secret-scanner $STAGED_FILES

RESULT=$?
if [ $RESULT -ne 0 ]; then
  echo ""
  echo "シークレット漏洩の疑いが検出されました。"
  echo "上記の指摘を確認してからコミットしてください。"
  echo "問題ない場合は: git commit --no-verify"
  exit 1
fi

exit 0
```

```bash
chmod +x .git/hooks/pre-commit
```

### GitHub Actionsへの組み込み

PRごとにCIでもスキャンを実行する設定です。

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scanner

on: [pull_request, push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 全履歴を取得（履歴スキャンのため）

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Secret Scanner
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: claude /secret-scanner . --fail-on-critical
```

`--fail-on-critical` オプションを付けると、CRITICALが検出された場合にexitコード1でCIを失敗させます。

---

## まとめ

`/secret-scanner` を使うことで、以下が自動化できます。

- ハードコードされたAWS/GitHub/Stripe等のキー検出
- 偽陽性の抑制（テストコード、プレースホルダー、低エントロピー値の除外）
- Git履歴への混入警告と対処ガイダンス
- pre-commitフック・CI/CDへの統合

シークレット漏洩はインシデントが発生してから対処するのでは遅い。コミット前の検出習慣が最も低コストな防御線です。

---

## Security Packについて

この記事で紹介した `/secret-scanner` は、**Security Pack**（¥1,480）に含まれています。

- `/security-audit` — OWASP Top 10 自動診断
- `/secret-scanner` — シークレット漏洩検出
- `/deps-check` — 依存ライブラリCVE検出

→ [PromptWorksで購入する](https://prompt-works.jp)

*みょうが (@myougaTheAxo) — セキュリティ重視のClaudeエンジニア。*
