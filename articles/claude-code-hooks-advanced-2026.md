---
title: "Claude Code Hooksの実践ガイド - 自動フォーマット・セキュリティガード・テスト連動"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "ai", "hooks", "devops", "コード品質"]
published: true
---

## Hooksとは

Claude Codeの**Hooks**は、AIがツール（コード書き込み・bash実行など）を実行する前後に自動でスクリプトを走らせる機能だ。

設定することで：
- **コードを書いた後** → 自動フォーマット
- **bashコマンド実行前** → 危険なコマンドをブロック
- **ファイル保存後** → lintチェック・セキュリティスキャン・テスト実行

人間がいちいち「フォーマットして」と言わなくても自動で品質が担保される。

---

## 設定方法

`.claude/settings.json`（プロジェクト固有）または `~/.claude/settings.json`（全プロジェクト共通）に書く。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "python .claude/hooks/guard_bash.py"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "python .claude/hooks/auto_format.py",
          "timeout": 30
        }]
      }
    ]
  }
}
```

---

## Hook 1: 危険なコマンドのガード（PreToolUse）

### 仕組み

`PreToolUse`フックは終了コード`2`を返すとツール実行をブロックできる。

```python
# .claude/hooks/guard_bash.py
import json
import sys

data = json.load(sys.stdin)
tool_name = data.get("tool_name", "")
tool_input = data.get("tool_input", {})

if tool_name != "Bash":
    sys.exit(0)

command = tool_input.get("command", "")

# 絶対にブロックするコマンド
HARD_BLOCK = [
    "rm -rf /",
    "rm -rf ~",
    "dd if=/dev/zero",
    ":(){ :|:& };:",  # Fork bomb
    "DROP DATABASE",
    "git push --force origin main",
    "git push --force origin master",
]

for pattern in HARD_BLOCK:
    if pattern in command:
        print(f"[BLOCKED] 禁止コマンド: {pattern}", file=sys.stderr)
        sys.exit(2)

# 要注意コマンドは警告のみ（ブロックしない）
WARN_PATTERNS = [
    "git push --force",
    "DROP TABLE",
    "chmod 777",
]

for pattern in WARN_PATTERNS:
    if pattern in command:
        print(f"[WARN] 要注意コマンド: {pattern}", file=sys.stderr)

sys.exit(0)
```

---

## Hook 2: 自動フォーマット（PostToolUse）

### 言語別フォーマッター自動選択

```python
# .claude/hooks/auto_format.py
import json
import subprocess
import sys
from pathlib import Path

data = json.load(sys.stdin)
file_path = data.get("tool_input", {}).get("file_path", "")

if not file_path:
    sys.exit(0)

path = Path(file_path)

FORMATTERS = {
    ".py": ["ruff", "format", "--quiet"],
    ".ts": ["npx", "prettier", "--write"],
    ".tsx": ["npx", "prettier", "--write"],
    ".js": ["npx", "prettier", "--write"],
    ".jsx": ["npx", "prettier", "--write"],
    ".go": ["gofmt", "-w"],
    ".rs": ["rustfmt"],
    ".kt": ["ktfmt"],
    ".java": ["google-java-format", "-i"],
}

formatter = FORMATTERS.get(path.suffix)
if formatter:
    subprocess.run([*formatter, str(path)], capture_output=True)

sys.exit(0)
```

---

## Hook 3: シークレット検出（PostToolUse）

コードにAPIキーが混入していないか自動チェックする。

```python
# .claude/hooks/scan_secrets.py
import json
import re
import sys

data = json.load(sys.stdin)
file_path = data.get("tool_input", {}).get("file_path", "")

SKIP_EXTENSIONS = {".jpg", ".png", ".gif", ".pdf", ".zip", ".epub"}
SKIP_DIRS = {"node_modules", ".git", "__pycache__", ".mypy_cache"}

if not file_path:
    sys.exit(0)

from pathlib import Path
p = Path(file_path)

if p.suffix in SKIP_EXTENSIONS:
    sys.exit(0)

if any(part in SKIP_DIRS for part in p.parts):
    sys.exit(0)

SECRET_PATTERNS = {
    "Anthropic API Key": r"sk-ant-api\d{2}-[a-zA-Z0-9_-]{86}",
    "AWS Access Key": r"AKIA[0-9A-Z]{16}",
    "GitHub PAT": r"ghp_[a-zA-Z0-9]{36}",
    "Stripe Key": r"sk_(live|test)_[a-zA-Z0-9]{24}",
    "OpenAI Key": r"sk-[a-zA-Z0-9]{48}",
    "Google API Key": r"AIza[0-9A-Za-z\-_]{35}",
}

EXCLUDE_PATTERNS = [
    r"YOUR_KEY_HERE", r"REPLACE_ME", r"example", r"placeholder",
    r"xxxx", r"test_", r"spec_", r"mock_",
]

try:
    content = p.read_text(encoding="utf-8", errors="replace")
except Exception:
    sys.exit(0)

for name, pattern in SECRET_PATTERNS.items():
    matches = re.findall(pattern, content)
    for match in matches:
        # 偽陽性フィルタ
        if any(re.search(ex, match, re.I) for ex in EXCLUDE_PATTERNS):
            continue
        print(f"[SECRET WARNING] {name} が検出されました: {match[:20]}...", file=sys.stderr)
        print(f"  ファイル: {file_path}", file=sys.stderr)
        # 警告のみ。ブロックする場合は sys.exit(2)

sys.exit(0)
```

---

## Hook 4: テスト自動実行（PostToolUse）

実装ファイルが変更されたとき、対応するテストを自動実行する。

```python
# .claude/hooks/run_related_tests.py
import json
import subprocess
import sys
from pathlib import Path

data = json.load(sys.stdin)
file_path = data.get("tool_input", {}).get("file_path", "")

if not file_path:
    sys.exit(0)

p = Path(file_path)

# src/ 内の Python ファイルのみ対象
if "src" not in p.parts or p.suffix != ".py" or p.name.startswith("test_"):
    sys.exit(0)

# 対応テストファイルを探す
test_candidates = [
    Path("tests") / f"test_{p.name}",
    Path("tests") / p.parent.name / f"test_{p.name}",
]

test_path = next((t for t in test_candidates if t.exists()), None)

if not test_path:
    sys.exit(0)

result = subprocess.run(
    ["pytest", str(test_path), "-q", "--no-header", "--tb=short"],
    capture_output=True, text=True, timeout=60
)

print(result.stdout)
if result.returncode != 0:
    print(f"[TEST FAILED] {test_path}", file=sys.stderr)
    print(result.stderr, file=sys.stderr)

sys.exit(0)
```

---

## 利用可能な環境変数

```bash
CLAUDE_PROJECT_DIR       # プロジェクトルート
CLAUDE_TOOL_NAME         # 実行ツール名 (Bash, Write, Edit, ...)
CLAUDE_TOOL_INPUT_FILE_PATH   # 操作ファイルパス (Write/Edit時)
CLAUDE_TOOL_INPUT_COMMAND     # 実行コマンド (Bash時)
```

---

## フックの管理をシンプルに保つコツ

### 1. フックは軽くする

重い処理をフックに入れると、Claude Codeの動作が遅くなる。

- フォーマット: OK（速い）
- ユニットテスト: OK（数秒以内なら）
- E2Eテスト: NG（遅すぎる）
- ビルド全体: NG（遅すぎる）

### 2. `timeout` を必ず設定する

```json
{
  "type": "command",
  "command": "python .claude/hooks/run_tests.py",
  "timeout": 30
}
```

タイムアウトがないと、フックが無限にハングした場合にClaude Codeが止まる。

### 3. エラーを握りつぶさない

```python
try:
    result = subprocess.run(...)
except Exception as e:
    print(f"Hook error: {e}", file=sys.stderr)
    sys.exit(0)  # エラーでも続行（ブロックしない）
```

フックのバグでClaude Code全体が止まらないように、エラー時は`sys.exit(0)`で続行する設計にしておく。

---

## まとめ

| Hook | タイミング | 主な用途 |
|------|-----------|---------|
| PreToolUse(Bash) | コマンド実行前 | 危険コマンドブロック |
| PostToolUse(Write/Edit) | ファイル書き込み後 | フォーマット、シークレットスキャン |
| PostToolUse(Write) | 新規ファイル作成後 | テスト自動実行 |

Hooksを一度設定してしまえば、Claude Codeとの協業品質が恒常的に向上する。設定コスト（数時間）に対するリターンは大きい。

---

*Security Pack（¥1,480）はシークレットスキャン・OWASP診断・CVEチェックを自動化します。Code Review Pack（¥980）はコードレビュー・リファクタリング・テスト生成を自動化します。*

*[PromptWorks](https://prompt-works.jp) で販売中。*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
