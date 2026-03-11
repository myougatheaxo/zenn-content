---
title: "DockerプロジェクトでのClaude Code活用術"
emoji: "🐳"
type: "tech"
topics: ["claudecode", "docker", "devcontainer", "開発効率化", "cli"]
published: true
---

## はじめに

Dockerを使う開発環境でClaude Codeを使う際、いくつかのポイントを押さえると格段に効率が上がる。

---

## CLAUDE.mdにDocker情報を書く

```markdown
## Docker環境
- 開発: `docker compose up -d`
- アプリ: http://localhost:8000
- DB: PostgreSQL (localhost:5432)
- Redis: localhost:6379
- ログ確認: `docker compose logs -f app`

## よく使うコマンド
- コンテナ内でコマンド実行: `docker compose exec app <command>`
- DBマイグレーション: `docker compose exec app alembic upgrade head`
- テスト実行: `docker compose exec app pytest -v`
- コンテナ再ビルド: `docker compose up -d --build`
```

これを書いておくと「DBマイグレーションを実行して」という指示が正しいdocker composeコマンドを生成する。

---

## Dev Containerとの組み合わせ

`.devcontainer/devcontainer.json` を使うと、Claude CodeをContainer内で実行できる。

```json
{
  "name": "my-app",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "app",
  "workspaceFolder": "/workspace",
  "features": {
    "ghcr.io/devcontainers/features/python:1": {"version": "3.12"},
    "ghcr.io/devcontainers/features/node:1": {"version": "20"}
  },
  "postCreateCommand": "pip install -e '.[dev]'",
  "customizations": {
    "vscode": {
      "extensions": ["anthropic.claude-code"]
    }
  }
}
```

---

## Hooksでコンテナ内操作を自動化

Docker環境では、ファイル編集後にコンテナ内で型チェックを実行したい場合がある。

```python
# .claude/hooks/docker_typecheck.py
import json
import subprocess
import sys

data = json.load(sys.stdin)
file_path = data.get("tool_input", {}).get("file_path", "")

if not file_path or not file_path.endswith(".py"):
    sys.exit(0)

# コンテナが起動していない場合はスキップ
check = subprocess.run(
    ["docker", "compose", "ps", "-q", "app"],
    capture_output=True, text=True
)
if not check.stdout.strip():
    sys.exit(0)

# コンテナ内でmypy実行
result = subprocess.run(
    ["docker", "compose", "exec", "-T", "app", "mypy", file_path],
    capture_output=True, text=True, timeout=30
)

if result.returncode != 0:
    print(result.stdout, file=sys.stderr)
    print(result.stderr, file=sys.stderr)

sys.exit(0)
```

---

## MCPでDockerログをリアルタイム確認

MCP Filesystemサーバーを使うと、Dockerのログファイルを自然言語で確認できる。

```bash
claude mcp add --transport stdio fs -- \
  npx @modelcontextprotocol/server-filesystem /var/log
```

```
「アプリのログで最後の10分間のエラーを見せて」
→ Claude Codeがログファイルを検索・分析
```

---

## Docker Compose環境のデバッグ

「なぜかコンテナが起動しない」という状況のデバッグに有効なプロンプト：

```
docker compose upが以下のエラーで失敗します：
[エラーメッセージ]

docker-compose.yml:
[ファイル内容]

考えられる原因と確認ポイントを教えてください。
```

---

## Dockerfileのベストプラクティスチェック

```
このDockerfileをレビューして、セキュリティとビルド効率の観点から問題点を指摘してください：

[Dockerfileの内容]
```

よく指摘されるポイント：
- `latest`タグの使用（バージョン固定すべき）
- rootユーザーでの実行（非rootユーザーを作るべき）
- 不要なファイルのコピー（.dockerignoreが必要）
- レイヤーのキャッシュ非効率（COPY --before RUN）

---

## まとめ

DockerプロジェクトでのClaude Code活用：

1. **CLAUDE.md** にDocker構成・よく使うコマンドを書く
2. **Dev Container** でClaude Codeをコンテナ内で実行
3. **Hooks** でコンテナ内の型チェックを自動化
4. **MCP** でログ確認を自然言語化

---

*Claude Codeをチームの開発標準にするための設定テンプレートは **Code Review Pack（¥980）** に含まれています。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
