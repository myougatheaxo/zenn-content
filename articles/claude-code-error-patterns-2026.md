---
title: "Claude Codeで一貫したエラーハンドリングを設計する"
emoji: "🚨"
type: "tech"
topics: ["claudecode", "typescript", "errorhandling", "設計", "nodejs"]
published: true
---

## はじめに

指示がないと、Claude Codeのエラーハンドリングはファイルごとにバラバラになる。ある関数ではthrow、別の関数ではnullを返す、またある関数ではエラーを握り潰す——という状態だ。

CLAUDE.mdでパターンを統一する。

---

## CLAUDE.mdにエラーハンドリングパターンを書く

```markdown
## エラーハンドリング

### カスタムエラークラス（src/errors/AppError.ts）
- AppError（基底クラス）: code, message, httpStatus
- ValidationError (400) — 入力バリデーション失敗
- NotFoundError (404) — リソースが見つからない
- AuthorizationError (403) — 権限不足
- ExternalServiceError (502) — 外部APIエラー

### 禁止パターン
- catch(e) {} — エラーを握り潰す
- エラーを示すためにnullを返す（throw or Result型を使う）
- ユーザー向けメッセージに内部詳細（スタックトレース等）を含める
- console.error だけしてre-throwしない

### パターン
- Service層はResult<T, AppError>を返す（throwしない）
- Controller層はResultをアンラップしてHTTPレスポンスに変換
- 未処理エラーはグローバルエラーハンドラー（src/middleware/errorHandler.ts）に伝播
```

---

## カスタムエラークラスの生成

```
このプロジェクトのカスタムエラークラス階層を生成してください。

要件：
- BaseのAppError（code, message, httpStatus付き）
- ValidationError (400)
- NotFoundError (404)
- AuthorizationError (403)
- ExternalServiceError (502)
- JSON直列化可能
- TypeScriptの型定義付き

保存先：src/errors/AppError.ts
```

---

## Resultパターンの適用

Serviceでthrowしない設計：

```typescript
type Result<T, E = AppError> =
  | { ok: true; value: T }
  | { ok: false; error: E };

const ok = <T>(value: T): Result<T> => ({ ok: true, value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });
```

Service：

```typescript
async function getUser(id: string): Promise<Result<User>> {
  const user = await db.user.findUnique({ where: { id } });
  if (!user) return err(new NotFoundError(`User ${id} not found`));
  return ok(user);
}
```

Controller：

```typescript
const result = await getUser(req.params.id);
if (!result.ok) {
  if (result.error instanceof NotFoundError) {
    return res.status(404).json({ error: result.error.message });
  }
  return res.status(500).json({ error: "Internal server error" });
}
return res.json(result.value);
```

---

## グローバルエラーハンドラーを生成させる

```
Expressのグローバルエラーハンドラーを生成してください。

要件：
- AppErrorサブクラスはhttpStatusを使ってレスポンスする
- PrismaエラーをマッピングL P2002=409, P2025=404
- Zodバリデーションエラーを400に変換
- 本番環境（NODE_ENV=production）ではスタックトレースを返さない
- 5xxエラーはlogger.ts（error level）で記録
- レスポンス形式: { error: string, code?: string }
```

---

## エラーハンドリングを検出するHook

```python
# .claude/hooks/check_errors.py
import json, re, sys

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
fp = data.get("tool_input", {}).get("file_path", "")

if not content or not fp.endswith((".ts", ".js")):
    sys.exit(0)

bad_patterns = [
    (r'catch\s*\([^)]*\)\s*\{\s*\}', "空のcatchブロック（エラー握り潰し）"),
    (r'catch\s*\([^)]*\)\s*\{\s*return null', "エラー時にnullを返却"),
    (r'catch\s*\([^)]*\)\s*\{\s*return undefined', "エラー時にundefinedを返却"),
]

for pattern, msg in bad_patterns:
    if re.search(pattern, content, re.DOTALL):
        print(f"[ERROR] {msg}", file=sys.stderr)

sys.exit(0)
```

---

## エラーテストの生成

```
このService関数のエラーハンドリングをテストするコードを生成してください。

テストケース：
- 正常ケース
- NotFoundError（HTTP 404を確認）
- ValidationError（HTTP 400を確認）
- DBエラー（HTTP 500、汎用メッセージを確認）

[関数をここに貼る]
```

---

## まとめ

Claude Codeで一貫したエラーハンドリング：

1. **CLAUDE.md** にカスタムエラークラスとルールを定義
2. **Resultパターン** でService層のthrowを禁止
3. **グローバルハンドラー** で全エラーを集約
4. **Hooks** でエラー握り潰しパターンを検出
5. **テスト** でエラーケースを網羅

---

*`/code-review` スキルがエラーハンドリングの問題を自動検出します。**Code Review Pack（¥980）** はPromptWorksで購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
