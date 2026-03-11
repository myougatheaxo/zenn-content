---
title: "Claude Code × MCP：自然言語でデータベース・GitHub・Slackを操作する"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "mcp", "github", "sqlite", "自動化"]
published: true
---

## MCPとは何か

MCP（Model Context Protocol）は、Claude Codeが外部ツールに接続するための標準プロトコルだ。

これを使うと：
- 「usersテーブルを見せて」→ SQLiteを直接クエリ
- 「このIssueをクローズして」→ GitHubのAPIを呼ぶ
- 「#devチャンネルに投稿して」→ Slackに送信

このような外部操作を、Claudeが自然言語で実行できるようになる。

---

## セットアップ: CLIコマンド一発

MCPサーバーの追加は簡単だ。

```bash
# SQLite接続
claude mcp add --transport stdio sqlite-db -- \
  npx @modelcontextprotocol/server-sqlite ./data/app.sqlite3

# GitHub連携
claude mcp add --transport stdio github -- \
  --env GITHUB_TOKEN=ghp_xxxxx \
  npx @modelcontextprotocol/server-github

# Slack連携
claude mcp add --transport stdio slack -- \
  --env SLACK_BOT_TOKEN=xoxb-xxxxx \
  npx @modelcontextprotocol/server-slack
```

設定ファイル（`.mcp.json`）で管理することもできる。

```json
{
  "mcpServers": {
    "app-db": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "./data/app.sqlite3"]
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

---

## 実際の操作例

### SQLiteデータベース操作

```
# Claude Codeに対して:
「usersテーブルの全カラムを見せて。
 その後、30日以上ログインしていないユーザーを抽出して」
```

Claude Codeは自動的にSQLを生成してクエリを実行し、結果を返す。人間がSQL文を書く必要がない。

```sql
-- 自動生成されるSQL（例）
SELECT u.id, u.email, u.last_login
FROM users u
WHERE u.last_login < datetime('now', '-30 days')
ORDER BY u.last_login ASC;
```

### GitHub Issue管理

```
# Claude Codeに対して:
「優先度Highのオープンなバグ一覧を見せて。
 その中で担当者が未設定のIssueに自分（myougatheaxo）をアサインして」
```

MCP経由でGitHub APIが呼ばれ、フィルタリングとアサインが自動実行される。

### Slack連携

```
# Claude Codeに対して:
「デプロイが完了した旨を#deploymentチャンネルに投稿して。
 バージョン番号はpackage.jsonから読んで」
```

Claude Codeがpackage.jsonを読み→Slack APIにPOSTする、という2ステップを自動的にこなす。

---

## よく使うMCPサーバー一覧

| サーバー名 | npm パッケージ | 主な用途 |
|-----------|--------------|---------|
| sqlite | @modelcontextprotocol/server-sqlite | SQLite操作 |
| github | @modelcontextprotocol/server-github | Issue/PR/ブランチ操作 |
| slack | @modelcontextprotocol/server-slack | チャンネル投稿・検索 |
| postgres | @modelcontextprotocol/server-postgres | PostgreSQL操作 |
| fetch | @modelcontextprotocol/server-fetch | Web取得 |
| brave-search | @modelcontextprotocol/server-brave-search | Web検索 |
| filesystem | @modelcontextprotocol/server-filesystem | ファイル操作の拡張 |

公式のMCPサーバーリストは GitHub `modelcontextprotocol/servers` で確認できる。

---

## カスタムMCPサーバーの作り方

既存のサーバーで対応できない場合は、自作できる。

```python
# Python製MCPサーバーの最小構成
from mcp.server import Server
from mcp.server.stdio import stdio_server

server = Server("my-custom-server")

@server.list_tools()
async def list_tools():
    return [
        {
            "name": "query_database",
            "description": "社内DBにクエリを実行する",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "実行するSQL"}
                },
                "required": ["sql"]
            }
        }
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "query_database":
        # 実際のDB操作
        result = execute_internal_db(arguments["sql"])
        return [{"type": "text", "text": str(result)}]

async def main():
    async with stdio_server() as streams:
        await server.run(*streams)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

これを `claude mcp add --transport stdio my-server -- python my_server.py` で登録すれば使える。

---

## セキュリティ上の注意点

MCPは強力なツールだが、セキュリティリスクもある。

### 1. 認証情報の管理

MCPサーバーのトークンは環境変数で渡す。設定ファイルに直書きしない。

```bash
# OK: 環境変数から読む
export GITHUB_TOKEN=ghp_xxxxx
claude mcp add --transport stdio github -- \
  --env GITHUB_TOKEN=${GITHUB_TOKEN} ...

# NG: 直書き
claude mcp add --transport stdio github -- \
  --env GITHUB_TOKEN=ghp_xxxxx ...  # トークンが設定ファイルに残る
```

### 2. 最小権限の原則

GitHubトークンは `repo` スコープのみ付与する（`admin:org` などは不要）。

### 3. ローカルファイルシステムの制限

filesystemサーバーを使う場合は、アクセス許可するパスを明示的に制限する。

```json
{
  "mcpServers": {
    "local-files": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-filesystem",
        "./src",
        "./docs"
      ]
    }
  }
}
```

ルートディレクトリ全体（`/`）を許可するのは危険だ。

---

## まとめ

MCPを使うと、Claude Codeは「コードを書くだけのツール」から「開発インフラ全体を操作できるAIエージェント」に変わる。

設定コストは低い（CLIコマンド1〜2行）のに、作業効率への影響は大きい。

特に「DB操作 + GitHub Issue管理 + デプロイ通知」の組み合わせは、日常的な開発作業の相当部分を自動化できる。

---

Claude Codeをさらに活用するカスタムスキルをPromptWorksで販売中：

- **Security Pack（¥1,480）** → /security-audit, /secret-scanner, /deps-check
- **Code Review Pack（¥980）** → /code-review, /refactor-suggest, /test-gen

[prompt-works.jp](https://prompt-works.jp)

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。Claude Codeの実践的な使い方を発信中。*
