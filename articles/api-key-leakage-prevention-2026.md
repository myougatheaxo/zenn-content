---
title: "APIキー漏洩を防ぐ5つの実践テクニック【Claude Codeで自動チェック】"
emoji: "🔐"
type: "tech"
topics: ["ClaudeCode", "Security", "Python", "Git", "DevOps"]
published: true
---

## なぜAPIキー漏洩が起きるのか

GitHubには毎日数千件のAPIキーが誤ってコミットされています。原因のほとんどはシンプルです。

- ハードコードしたまま `git add .` してしまった
- `.gitignore` の設定ミス
- ログやエラーメッセージにキーを含めてしまった
- コードを共有するときに `.env` ファイルごと渡した

漏洩したキーはクローラーによって数分以内に検出され、不正利用されます。AWSキーの場合、数時間で数十万円の請求が来るケースも報告されています。

## テクニック1: .envファイルで環境変数を管理する

絶対にコードにキーを直書きしてはいけません。

```python
# 悪い例
import openai
openai.api_key = "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# 正しい例
import os
from dotenv import load_dotenv

load_dotenv()
openai.api_key = os.getenv("OPENAI_API_KEY")
```

`.env` ファイルの例：

```
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
DATABASE_URL=postgresql://user:pass@localhost/mydb
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
```

## テクニック2: .gitignoreを正しく設定する

プロジェクト作成時に必ず `.gitignore` を設定します。

```gitignore
# 環境変数ファイル
.env
.env.local
.env.*.local
*.env

# 認証情報ファイル
credentials.json
token.json
*_token.txt
*.pem
*.key

# IDE・OS固有ファイル
.DS_Store
Thumbs.db
.idea/
.vscode/settings.json
```

すでにコミットしてしまったファイルは、追加するだけでは不十分です。

```bash
# すでにトラッキングされているファイルをgitの管理から外す
git rm --cached .env
git commit -m "Remove .env from tracking"
```

## テクニック3: git-secretsで自動ブロックする

[git-secrets](https://github.com/awslabs/git-secrets) は、コミット時にキーのパターンを検出してブロックするツールです。

```bash
# インストール（macOS）
brew install git-secrets

# リポジトリに設定
git secrets --install
git secrets --register-aws  # AWSパターンを登録

# 独自パターンを追加
git secrets --add 'sk-[a-zA-Z0-9]{48}'  # OpenAI
git secrets --add 'xoxb-[0-9]+-[0-9]+-[a-zA-Z0-9]+'  # Slack
```

誤ってキーをコミットしようとすると：

```
[ERROR] Matched one or more prohibited patterns
Possible mitigations:
- Mark false positives as allowed using: git config --add secrets.allowed ...
```

## テクニック4: エントロピー分析で未知のキーも検出する

パターンに依存しない検出方法として、シャノンエントロピーを使います。ランダムな文字列（=キー）は高エントロピー値を持ちます。

```python
import math
import re

def shannon_entropy(s):
    freq = {}
    for c in s:
        freq[c] = freq.get(c, 0) + 1
    entropy = 0
    for count in freq.values():
        p = count / len(s)
        entropy -= p * math.log2(p)
    return entropy

def scan_file(filepath):
    with open(filepath, 'r', errors='ignore') as f:
        for i, line in enumerate(f, 1):
            # 長い英数字文字列を抽出
            tokens = re.findall(r'[A-Za-z0-9+/]{20,}', line)
            for token in tokens:
                entropy = shannon_entropy(token)
                if entropy > 4.5:  # 閾値: 高エントロピー
                    print(f"L{i}: High entropy string detected: {token[:10]}...")
```

## テクニック5: /secret-scannerスキルで自動化する

Claude Codeの `/secret-scanner` スキルを使えば、コミット前に全ファイルをスキャンできます。

```bash
# スキャン実行
claude /secret-scanner src/
```

スキル設定（`.claude/commands/secret-scanner.md`）：

```markdown
以下のルールでシークレット検出スキャンを実施してください。

## 検出対象
1. APIキー（OpenAI, AWS, GitHub, Slack等）のパターンマッチ
2. シャノンエントロピー4.5以上の長い文字列
3. .envファイルがgitトラッキングされていないか確認
4. パスワードのハードコード（password=, passwd=, secret=）

## 出力フォーマット
- 深刻度: HIGH / MEDIUM / LOW
- ファイル名:行番号
- 検出内容
- 対処方法
```

CI/CDに組み込む例（GitHub Actions）：

```yaml
name: Secret Scan
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run secret scan
        run: |
          pip install truffleHog
          truffleHog --regex --entropy=True .
```

## 漏洩してしまったときの緊急対応

もし漏洩してしまった場合、以下の順番で対応します。

1. **即座にキーを無効化** — APIプロバイダーのダッシュボードで削除
2. **新しいキーを発行** — 別のシークレットマネージャーに保存
3. **履歴から削除** — `git filter-branch` または `BFG Repo Cleaner`
4. **影響範囲を確認** — アクセスログで不正利用を調査

```bash
# BFGでファイルをgit履歴から完全削除
brew install bfg
bfg --delete-files .env
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push --force
```

## まとめ

| テクニック | 効果 | 難易度 |
|-----------|------|--------|
| .env + dotenv | ハードコード防止 | 低 |
| .gitignore設定 | 誤コミット防止 | 低 |
| git-secrets | 自動ブロック | 中 |
| エントロピー分析 | 未知パターン検出 | 中 |
| /secret-scanner | CI/CD統合 | 低 |

APIキー漏洩の90%は基本的なミスから生じます。まず `.env` と `.gitignore` を徹底するだけで大半のリスクを排除できます。

---

> **Security Pack（¥1,480）** — `/secret-scanner`・`/security-audit`・`/deps-check` の3スキルセット。APIキー漏洩検出からOWASP準拠チェックまでカバー。
>
> 👉 [note.com/myougatheaxo](https://note.com/myougatheaxo)
