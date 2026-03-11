---
title: "Claude CodeでOpenAPIドキュメントを自動生成する：コードファーストAPI設計"
emoji: "📄"
type: "tech"
topics: ["claudecode", "openapi", "swagger", "typescript", "api設計"]
published: true
---

## はじめに

APIの実装とドキュメントが乖離するのはよくある問題だ。Claude Codeを使ってコードから自動的にOpenAPIドキュメントを生成し、常にドキュメントを最新に保つパターンを紹介する。

---

## CLAUDE.mdにAPIドキュメントルールを書く

```markdown
## APIドキュメントルール

### OpenAPI仕様
- バージョン: OpenAPI 3.1
- ドキュメントファイル: docs/openapi.yaml
- コードファースト: コードからOpenAPIを自動生成（手書き禁止）

### コードアノテーション
- TypeScriptの型定義がスキーマになる
- JSDocコメントでdescriptionを記述
- @example タグで例を記載

### エンドポイント要件
- 全エンドポイントにsummaryとdescription必須
- 全レスポンスに型定義必須（200/400/401/500）
- ページネーションは標準フォーマット統一
  - リクエスト: page, limit (max 100)
  - レスポンス: { data, total, page, limit }

### ツール
- tsoa（TypeScriptデコレーターから自動生成）または
- zod-to-openapi（Zodスキーマから生成）
```

---

## zodとzod-to-openapi連携の設定生成

```
zod-to-openapiを使ってZodスキーマからOpenAPI仕様を自動生成してください。

既存のZodスキーマ:
- CreateUserSchema: { email: string, name: string, role: 'admin' | 'user' }
- UserSchema: { id: string, email: string, name: string, role: string, createdAt: string }

要件：
- OpenAPI 3.1
- スキーマにdescription・example・deprecatedを付ける
- 認証: BearerToken (JWT)
- エラーレスポンス: { error: string, details?: object }

生成先: src/docs/openapi.ts → docs/openapi.yaml
```

---

## 生成されるOpenAPI定義の例

```typescript
// src/docs/schemas.ts
import { extendZodWithOpenApi } from 'zod-to-openapi';
import { z } from 'zod';

extendZodWithOpenApi(z);

export const CreateUserSchema = z
  .object({
    email: z
      .string()
      .email()
      .openapi({ description: 'ユーザーのメールアドレス', example: 'user@example.com' }),
    name: z
      .string()
      .min(1)
      .max(100)
      .openapi({ description: '表示名', example: '山田太郎' }),
    role: z
      .enum(['admin', 'user'])
      .openapi({ description: 'ユーザーロール', example: 'user' }),
  })
  .openapi('CreateUserRequest');

export const UserSchema = z
  .object({
    id: z.string().uuid().openapi({ description: 'ユーザーID' }),
    email: z.string().email(),
    name: z.string(),
    role: z.enum(['admin', 'user']),
    createdAt: z.string().datetime(),
  })
  .openapi('User');
```

---

## エンドポイントのOpenAPIアノテーション生成

```
以下のExpressエンドポイントにOpenAPIアノテーションを追加してください。

対象:
- POST /users（ユーザー作成）
- GET /users（ユーザー一覧、ページネーション付き）
- GET /users/:id（ユーザー詳細）
- PATCH /users/:id（部分更新）
- DELETE /users/:id（論理削除）

各エンドポイントに追加する内容:
- summary・description
- requestBody（Zodスキーマを参照）
- responses（200/201/400/401/403/404/500）
- security（JWT認証が必要なエンドポイント）
- tags（['Users']）
```

---

## Swagger UIの統合

```
Express アプリにSwagger UIを追加してください。

要件：
- パス: /api/docs
- OpenAPIファイル: docs/openapi.yaml を読み込む
- 本番環境では無効化（NODE_ENV=production）
- ダークテーマ設定
- APIキーをUIから試せるように設定

パッケージ: swagger-ui-express, js-yaml
```

```typescript
// 生成されるミドルウェア例
import swaggerUi from 'swagger-ui-express';
import YAML from 'js-yaml';
import fs from 'fs';

if (process.env.NODE_ENV !== 'production') {
  const openapiSpec = YAML.load(fs.readFileSync('./docs/openapi.yaml', 'utf8'));
  app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(openapiSpec, {
    customCss: '.swagger-ui .topbar { display: none }',
    swaggerOptions: { persistAuthorization: true },
  }));
}
```

---

## CI/CDでドキュメントの同期確認

```
OpenAPIドキュメントとコードの乖離を検出するCIステップを追加してください。

要件：
- ビルド時にOpenAPIを再生成
- 既存のdocs/openapi.yamlと差分がある場合はCIを失敗させる
- 差分がある場合は「`npm run generate:openapi`を実行してコミットしてください」というエラーメッセージを表示
```

---

## まとめ

Claude CodeでOpenAPIドキュメントを自動生成する：

1. **CLAUDE.md** にコードファースト・zod-to-openapiを標準ツールとして明記
2. **ZodスキーマにOpenAPIアノテーションを追加** させる
3. **エンドポイントのレスポンス型を網羅** させる（400/401/403/500全て）
4. **Swagger UI** をdoc環境だけに追加
5. **CIで同期確認** → ドキュメント漏れを自動検出

---

*APIドキュメントの漏れとスキーマ不整合を自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
