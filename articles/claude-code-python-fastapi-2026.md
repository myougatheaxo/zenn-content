---
title: "Python/FastAPIプロジェクトでClaude Codeを最大活用する設定ガイド"
emoji: "🐍"
type: "tech"
topics: ["claudecode", "python", "fastapi", "pydantic", "pytest"]
published: true
---

## はじめに

TypeScriptと同じくらい、PythonプロジェクトでもClaude Codeの設定次第で出力品質が大きく変わる。

この記事では、Python/FastAPI（Pydantic + SQLAlchemy + Pytest構成）での実践的な設定を紹介する。

---

## CLAUDE.md: Python/FastAPI向け設定

```markdown
# プロジェクト名: my-fastapi

## 技術スタック
- Python 3.12
- FastAPI 0.110 + Uvicorn
- SQLAlchemy 2.x (async) + Alembic
- Pydantic v2
- PostgreSQL 16

## コマンド
- サーバー起動: `uvicorn app.main:app --reload`
- テスト: `pytest -v`
- Lint: `ruff check . && mypy .`
- マイグレーション: `alembic upgrade head`

## アーキテクチャ (変更禁止)
- レイヤー: Router → Service → Repository
- Routerに直接DBアクセスを書かない
- Serviceにリクエスト/レスポンス型を持ち込まない
- 依存性注入はFastAPIのDependsを使う

## コーディングルール
- 型アノテーション必須（mypy strict対応）
- `Any`型は禁止（やむを得ない場合はコメントで理由明記）
- print()は本番コードに残さない（loguru使用）
- 全ての非同期関数はtry/exceptでエラーハンドリング
- 文字列フォーマットはf-stringを使う

## セキュリティルール
- SQLはSQLAlchemy ORM経由のみ（raw SQL禁止）
- 外部入力は全てPydanticでバリデーション
- シークレットは`python-dotenv`経由のみ（ハードコード禁止）
- ログにパスワード・トークン・個人情報を出力しない

## テスト規約
- テストファイル: `tests/test_*.py`
- カバレッジ目標: 80%以上
- 外部サービスは`pytest-mock`でモック
- DBテストは`pytest-asyncio` + テスト用DB使用

## よく参照するファイル
- DB接続: `app/db/session.py` (AsyncSessionシングルトン)
- 認証依存: `app/api/deps.py`
- 共通型: `app/schemas/`
- エラークラス: `app/core/exceptions.py`
```

---

## Hooks設定: 自動品質チェック

`.claude/settings.json` に書く。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "python .claude/hooks/py_quality.py",
          "timeout": 30
        }]
      }
    ]
  }
}
```

### Python品質チェックHook

```python
# .claude/hooks/py_quality.py
import json
import subprocess
import sys
from pathlib import Path

data = json.load(sys.stdin)
file_path = data.get("tool_input", {}).get("file_path", "")

if not file_path:
    sys.exit(0)

p = Path(file_path)
if p.suffix != ".py":
    sys.exit(0)

# 自動フォーマット
subprocess.run(["ruff", "format", str(p)], capture_output=True)
subprocess.run(["ruff", "check", "--fix", str(p)], capture_output=True)

# テストファイルじゃない場合、print()チェック
if "test" not in p.name and "app/" in str(p):
    content = p.read_text()
    if "print(" in content and "# allow-print" not in content:
        print(f"WARNING: print() in {p.name} - use loguru instead", file=sys.stderr)

sys.exit(0)
```

---

## SQLAlchemy 2.x asyncの活用

`CLAUDE.md` にasyncパターンを明示しておく。

```markdown
## SQLAlchemy 2.x ルール
- `AsyncSession`を使う（syncのSessionは禁止）
- クエリは`select()`スタイル（古いquery()スタイル禁止）
- リレーションは`selectinload()`でeager loading
- `AsyncSession`はContextManagerで使う
```

こうすると「Userモデルの記事一覧を取得して」という指示が正しいasyncコードを生成する。

---

## テスト生成: pytest + pytest-asyncio

`CLAUDE.md` にテストパターンを明示すると精度が上がる。

```markdown
## テスト詳細
- フレームワーク: pytest + pytest-asyncio
- asyncテスト: @pytest.mark.asyncio デコレータ必須
- DB: pytest-asyncio + テスト用SQLite in-memory
- Fixture: conftest.py に async_session, async_client を定義
- パターン: AAA（Arrange, Act, Assert）
```

生成されるテストの例：

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_user(async_client: AsyncClient):
    # Arrange
    payload = {"email": "test@example.com", "name": "Test User"}

    # Act
    response = await async_client.post("/api/v1/users", json=payload)

    # Assert
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == payload["email"]
    assert "id" in data
```

---

## Pydantic v2の型安全パターン

`CLAUDE.md` にPydantic v2のルールを書く。

```markdown
## Pydanticルール
- Pydantic v2を使う（v1のvalidator/@validatorは禁止）
- `model_validator`と`field_validator`を使う
- レスポンスはすべてBaseModelを継承
- `model_config = ConfigDict(from_attributes=True)` を使う
```

---

## よくあるPythonのミスをClaude Codeが防ぐ

CLAUDE.mdに制約を書いておくと、以下のパターンを自動的に回避する：

| アンチパターン | 防ぐ理由 |
|---|---|
| `eval()`/`exec()` | セキュリティリスク |
| `SELECT *` クエリ | N+1問題・不要データ取得 |
| 例外の握り潰し `except: pass` | デバッグ困難 |
| ミュータブルデフォルト引数 | バグの温床 |
| `asyncio.run()` をasync関数内で呼ぶ | デッドロック |

---

## まとめ

Python/FastAPIプロジェクトでClaude Codeを最大活用するための優先順位：

1. **CLAUDE.md** に技術スタック・制約・asyncパターンを書く（最優先）
2. **Hooks** でruff自動フォーマット・print()チェックを設定
3. **MCP** でDB確認・API動作確認を自然言語化
4. **カスタムスキル** でコードレビュー・テスト生成を自動化

---

*カスタムスキル（/code-review, /refactor-suggest, /test-gen）は **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
