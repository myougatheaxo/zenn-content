---
title: "TypeScript/Node.jsプロジェクトでClaude Codeを最大活用する設定ガイド"
emoji: "🔷"
type: "tech"
topics: ["claudecode", "typescript", "nodejs", "prisma", "vitest"]
published: true
---

## はじめに

Claude Codeはデフォルト状態でも優秀だが、TypeScriptプロジェクトに合わせて設定を作り込むと、出力品質が別次元になる。

この記事では、TypeScript/Node.js（Express + Prisma構成）での実践的な設定を紹介する。

---

## CLAUDE.md: 最も効果が高い設定

プロジェクトルートの `CLAUDE.md` に以下を書く。これがあるとないとでは、出力品質に明確な差が出る。

```markdown
# プロジェクト名: my-api

## 技術スタック
- Node.js 20 LTS + TypeScript 5.x (strict: true)
- Express 4 + Prisma ORM
- PostgreSQL 16

## コマンド
- テスト: `npm test` (Vitest使用)
- Lint: `npm run lint` (ESLint + Prettier)
- ビルド: `npm run build`
- DBマイグレーション: `npx prisma migrate dev`

## アーキテクチャ (変更禁止)
- レイヤー構成: Router → Controller → Service → Repository
- ControllerにDB操作を書かない
- ServiceにHTTP依存コードを書かない
- 依存性注入はコンストラクタ経由

## コーディングルール
- `any`型は原則禁止（使う場合はコメントで理由を明記）
- 本番コードに `console.log` を残さない（logger.tsを使う）
- 全ての非同期関数はtry/catchでエラーハンドリング
- マジックナンバーは定数化

## セキュリティルール
- SQLはPrisma経由のみ（raw文字列クエリ禁止）
- 外部入力は全てZodでバリデーション
- ログにPII（個人情報）を出力しない
- シークレットは必ず `process.env` 経由

## テスト規約
- テストファイル: `src/**/*.test.ts`
- カバレッジ目標: 80%以上
- 外部サービスは必ずモック（実API呼び出し禁止）
- テスト間で状態を共有しない（beforeEachでセットアップ）

## よく参照するファイル
- DB接続: `src/lib/db.ts` (Prismaシングルトン)
- 認証MW: `src/middleware/auth.ts`
- 共通型: `src/types/index.ts`
- エラークラス: `src/errors/AppError.ts`
```

---

## Hooks設定: 自動品質チェック

`.claude/settings.json` に書く。

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
          "command": "python .claude/hooks/ts_quality.py",
          "timeout": 30
        }]
      }
    ]
  }
}
```

### TypeScript品質チェックHook

```python
# .claude/hooks/ts_quality.py
import json
import subprocess
import sys
from pathlib import Path

data = json.load(sys.stdin)
file_path = data.get("tool_input", {}).get("file_path", "")

if not file_path:
    sys.exit(0)

p = Path(file_path)
if p.suffix not in (".ts", ".tsx"):
    sys.exit(0)

# 自動フォーマット
subprocess.run(["npx", "prettier", "--write", str(p)], capture_output=True)
subprocess.run(["npx", "eslint", "--fix", str(p)], capture_output=True)

# テストファイルじゃない場合、console.logチェック
if "test" not in p.name and "src" in str(p):
    content = p.read_text()
    if "console.log(" in content:
        print(f"WARNING: console.log in {p.name} - use logger.ts", file=sys.stderr)

sys.exit(0)
```

---

## Prisma連携のコツ

Prismaスキーマをよく参照するプロジェクトでは、CLAUDE.mdにスキーマの場所を明記しておく。

```markdown
## Prismaスキーマ
- ファイル: `prisma/schema.prisma`
- マイグレーション後は必ず `npx prisma generate` を実行
- クライアントのシングルトン: `src/lib/db.ts`
```

こうすると「UserモデルにrefreshTokenフィールドを追加して」という指示が正確に動く。

---

## テスト生成: /test-gen の活用

`/test-gen` スキル（Code Review Pack付属）を使うと、既存のサービス層のコードからVitestテストを自動生成できる。

```bash
/test-gen src/services/user.service.ts
```

生成されるテストの例：

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { UserService } from "./user.service";
import { prismaMock } from "../lib/__mocks__/db";

describe("UserService", () => {
  let service: UserService;

  beforeEach(() => {
    service = new UserService(prismaMock);
    vi.clearAllMocks();
  });

  describe("findById", () => {
    it("既存ユーザーのIDで正しいユーザーを返す", async () => {
      const mockUser = { id: "1", email: "test@example.com", name: "Test" };
      prismaMock.user.findUnique.mockResolvedValue(mockUser);

      const result = await service.findById("1");
      expect(result).toEqual(mockUser);
    });

    it("存在しないIDでnullを返す", async () => {
      prismaMock.user.findUnique.mockResolvedValue(null);

      const result = await service.findById("nonexistent");
      expect(result).toBeNull();
    });
  });
});
```

---

## よくあるTypeScriptのミスをClaude Codeが防ぐ

CLAUDE.mdに制約を書いておくと、Claude Codeが自動的に以下のパターンを回避する：

| アンチパターン | 防ぐ理由 |
|-------------|---------|
| `const x = value as any` | 型安全性の崩壊 |
| `user!.name` (non-null assertion) | 実行時エラーリスク |
| `JSON.parse()` の結果を型なしで使う | 実行時型エラー |
| `async function` のtry/catch漏れ | Unhandled rejection |
| `console.log` をコミット | ログ汚染・情報漏洩 |

---

## MCP連携: PrismaデータをClaude Codeから直接確認

MCPのSQLiteサーバー（またはPostgresサーバー）を使うと、開発中のデータを自然言語で確認できる。

```bash
# 開発DB（SQLite）の場合
claude mcp add --transport stdio dev-db -- \
  npx @modelcontextprotocol/server-sqlite ./dev.db
```

```
「最後の10件のユーザー登録を見せて」
→ SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
```

---

## まとめ

TypeScript/Node.jsプロジェクトでClaude Codeを最大活用するための優先順位：

1. **CLAUDE.md** に技術スタック・制約・よく使うファイルを書く（最優先）
2. **Hooks** で自動フォーマット・品質チェックを設定
3. **カスタムスキル** でコードレビュー・テスト生成を自動化
4. **MCP** でDB確認・GitHub操作を自然言語化

この4つを設定すれば、Claude Codeとの協業効率が大幅に上がる。

---

*カスタムスキル（/code-review, /refactor-suggest, /test-gen）は **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
