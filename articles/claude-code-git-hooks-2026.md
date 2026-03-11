---
title: "Claude Codeとgit hooksを組み合わせる：pre-commitで品質ゲートを作る"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "githooks", "precommit", "nodejs", "品質管理"]
published: true
---

## はじめに

Claude Codeが生成したコードをコミットする前に自動チェックをかけたい——git hooksとClaude Codeを組み合わせると、コミット時に品質ゲートが自動的に発動する。

---

## CLAUDE.mdにgit hooks方針を書く

```markdown
## Git Hooks方針

### pre-commit（必須チェック）
- lintエラーがある場合はコミット失敗
- TypeScriptのビルドエラーはコミット失敗
- console.log が本番コードに残っている場合は警告（--force で上書き可）

### commit-msg（メッセージ規約）
- Conventional Commits形式: feat/fix/docs/refactor/test/chore
- 例: feat(auth): add JWT refresh token logic
- スコープ必須: feat(scope): description

### pre-push
- テストスイートを実行
- カバレッジ70%未満で失敗

### セットアップ
- huskyで管理（.husky/ ディレクトリ）
- `npm run prepare` でフックをインストール
```

---

## huskyのセットアップを生成させる

```
huskyとlint-stagedを使ったgit hooksセットアップを生成してください。

要件：
- pre-commit: ESLint（変更ファイルのみ）+ TypeScript型チェック
- commit-msg: Conventional Commits形式の検証
- pre-push: テスト実行（npm test）
- Node.js 20 + TypeScript 5 + ESLint 8

生成するファイル:
- package.json の scripts / devDependencies 追記
- .husky/pre-commit
- .husky/commit-msg
- .husky/pre-push
- .lintstagedrc.json
```

---

## pre-commitフックの生成例

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🔍 pre-commit: 変更ファイルにlintを実行..."
npx lint-staged

echo "🔷 TypeScript型チェック..."
npx tsc --noEmit --skipLibCheck

echo "✅ pre-commit完了"
```

```json
// .lintstagedrc.json
{
  "*.{ts,tsx}": [
    "eslint --fix",
    "prettier --write"
  ],
  "*.{json,md,yml}": [
    "prettier --write"
  ]
}
```

---

## commit-msgフックでConventional Commitsを強制する

```
Conventional Commitsを強制するcommit-msgフックを生成してください。

要件：
- 形式: type(scope): description
- 有効なtype: feat, fix, docs, style, refactor, test, chore, perf, ci
- scopeは小文字英数字のみ
- descriptionは50文字以内
- BREAKING CHANGE対応
- エラーメッセージは日本語で表示

保存先: .husky/commit-msg
```

```bash
# 生成されるフック例
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

COMMIT_MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci)(\([a-z0-9-]+\))!?: .{1,50}$"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo "❌ コミットメッセージが規約に違反しています。"
  echo ""
  echo "正しい形式: type(scope): description"
  echo "例: feat(auth): JWTリフレッシュトークンを追加"
  echo ""
  echo "有効なtype: feat, fix, docs, style, refactor, test, chore, perf, ci"
  exit 1
fi
```

---

## Claude Code Hooksでコミット前にセキュリティスキャン

Claude Code自体のHooksを使って、ファイル書き込み時にセキュリティ問題を検出することもできる。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{"type": "command", "command": "python .claude/hooks/security_check.py"}]
      }
    ]
  }
}
```

```python
# .claude/hooks/security_check.py
import json, re, sys

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
fp = data.get("tool_input", {}).get("file_path", "")

if not fp or not fp.endswith((".ts", ".js", ".py")):
    sys.exit(0)

# ハードコードされたシークレットのパターン
PATTERNS = [
    (r'(?i)(password|passwd|pwd)\s*=\s*["\'][^"\']{8,}["\']', "ハードコードされたパスワード"),
    (r'(?i)api[_-]?key\s*=\s*["\'][a-zA-Z0-9]{20,}["\']', "ハードコードされたAPIキー"),
    (r'(?i)secret\s*=\s*["\'][^"\']{10,}["\']', "ハードコードされたシークレット"),
]

for pattern, message in PATTERNS:
    if re.search(pattern, content):
        print(f"[SECURITY] {message}を検出しました。環境変数に移してください。", file=sys.stderr)
        sys.exit(2)

sys.exit(0)
```

---

## pre-pushでカバレッジゲートを設ける

```
テストカバレッジが70%未満の場合にpushをブロックするpre-pushフックを生成してください。

要件：
- vitestのカバレッジレポートを使用
- JSON形式のレポートを解析
- ステートメントカバレッジが70%以上なら成功
- 失敗時はカバレッジレポートのサマリーを表示

保存先: .husky/pre-push
```

---

## まとめ

Claude Codeとgit hooksで品質ゲートを構築する：

1. **CLAUDE.md** にhooksの方針を明記（huskyで管理）
2. **pre-commit** でlint + 型チェックを自動実行
3. **commit-msg** でConventional Commitsを強制
4. **pre-push** でテストカバレッジゲートを設ける
5. **Claude Code Hooks** でファイル書き込み時にセキュリティスキャン

---

*git hooksとセキュリティスキャンを自動化するスキルは **Security Pack（¥1,480）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
