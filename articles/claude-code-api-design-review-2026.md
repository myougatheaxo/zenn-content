---
title: "Claude CodeでREST API設計のレビューとドキュメント自動生成"
emoji: "🤖"
type: "tech"
topics:
  - claudecode
  - api
  - restapi
  - openapi
  - 開発効率
published: true
---

## APIのどこをレビューするか

Claude CodeにAPIレビューを依頼する前に、「何をチェックするか」を明確にする。

主要なチェック項目:
1. **URLの一貫性** (リソース名、動詞の排除)
2. **HTTPメソッドの正しい使用** (GET/POST/PUT/PATCH/DELETE)
3. **ステータスコード** (201 vs 200, 404 vs 400等)
4. **エラーレスポンス形式** (一貫したエラースキーマ)
5. **ページネーション** (cursor vs offset, 命名規則)
6. **バージョニング** (/v1/, Accept-Version等)

---

## REST API設計レビューコマンド

`.claude/commands/api-review.md`:

```markdown
# REST API設計レビュー

$ARGUMENTS のAPIエンドポイントを以下の観点でレビュー:

## REST原則
- URLにリソース名(名詞)のみ使用しているか
- 適切なHTTPメソッドを使っているか
- ネストは2階層以内か (/users/:id/orders まで)

## ステータスコード
- GET: 200 (OK), 404 (not found)
- POST: 201 (created), 400 (bad request)
- PUT/PATCH: 200 (ok), 404 (not found)
- DELETE: 204 (no content), 404 (not found)

## エラーレスポンス
全エンドポイントで統一された形式か:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "説明",
    "details": [...]
  }
}
```

## 問題点を列挙して修正案を提示
```

---

## OpenAPIドキュメント自動生成

```bash
claude "src/routes/ 以下のExpressルートを解析して
OpenAPI 3.0形式のYAMLを生成。

含めるもの:
- 全エンドポイント (path, method, summary)
- リクエストボディのスキーマ (TypeScriptの型から)
- レスポンススキーマ
- 認証要件 (Bearer tokenが必要なもの)
- エラーレスポンス

出力先: docs/openapi.yaml"
```

---

## 実際のAPI設計問題例

```typescript
// ❌ 問題のあるAPI設計
router.post('/getUser', handler)          // 動詞がURL内
router.get('/users/deleteUser/:id', handler)  // DELETEを使うべき
router.post('/users/activate', handler)   // PATCH /users/:id/status が良い

// ✅ 修正後
router.get('/users/:id', handler)
router.delete('/users/:id', handler)
router.patch('/users/:id/status', handler)
```

---

## TypeScriptからOpenAPIスキーマを生成

```bash
claude "src/types/user.ts の User, CreateUserDTO, UpdateUserDTO を
OpenAPI componentsのスキーマに変換。

入力:
```typescript
interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
}
```

出力:
```yaml
components:
  schemas:
    User:
      type: object
      required: [id, email, name, createdAt]
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
```
"
```

---

## APIバージョニング戦略のレビュー

```bash
claude "現在のAPIバージョニング戦略を評価して。
比較対象:
1. URLパス (/v1/, /v2/)
2. クエリパラメータ (?version=1)
3. Acceptヘッダー (Accept: application/vnd.api+json; version=2)
4. カスタムヘッダー (X-API-Version: 2)

現在のプロジェクト規模と将来の保守性を考慮して推奨案を提示"
```

---

## ドキュメントとコードの同期

Hooksでコード変更時にドキュメントを自動更新:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo "$CLAUDE_TOOL_INPUT" | grep -q "routes/" && echo "[REMINDER] APIドキュメントの更新が必要です: docs/openapi.yaml" || true'"
          }
        ]
      }
    ]
  }
}
```

---

## まとめ

| タスク | プロンプト例 |
|--------|------------|
| APIレビュー | /api-review routes/users.ts |
| OpenAPI生成 | "src/routes/をOpenAPI YAMLに変換" |
| 型→スキーマ変換 | "TypeScript型をOpenAPIスキーマに" |
| バージョニング相談 | "現在の戦略を評価して推奨案を" |

APIドキュメントは書いた直後が一番正確。Hooksで変更を検知して、常に最新の状態を維持する仕組みを作る。


---

この記事の内容はnoteで公開しているClaude Code完全攻略ガイド（全7章）から抜粋したもの。MCPの構築方法、カスタムスキル設計、マルチエージェント構成まで網羅している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
