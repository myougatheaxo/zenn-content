---
title: "Claude CodeでAPI Contract-Firstを設計する：OpenAPI→TypeScript自動生成・Mock・バリデーション"
emoji: "📋"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "openapi", "testing"]
published: true
published_at: "2026-03-13 09:00"
---

## はじめに

「フロントエンドとバックエンドで型が合わない」——API Contract-FirstでOpenAPI仕様を先に書き、TypeScriptの型・バリデーション・Mockを自動生成する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにContract-First設計ルールを書く

```markdown
## API Contract-First設計ルール

### ワークフロー
1. OpenAPI 3.1仕様を先に書く
2. TypeScript型を自動生成（openapi-typescript）
3. Express/Fastifyのバリデーションを自動適用
4. Mockサーバー（Prism）でフロントエンドが先行開発可能

### 設計原則
- 仕様が唯一の真実（実装と仕様の乖離禁止）
- PRにOpenAPI変更を含める場合は変更履歴を記録
- バージョニング: URLパス（/api/v1/）

### バリデーション
- Request: Zodスキーマ（OpenAPIから生成）
- Response: Zodでランタイム検証（開発時のみ）
```

---

## Contract-First APIの生成

```
OpenAPI Contract-FirstのAPIシステムを設計してください。

要件：
- OpenAPI 3.1仕様定義
- TypeScript型自動生成
- Zodバリデーション
- Mockサーバー設定
- CI/CD統合（仕様と実装の整合性確認）

生成ファイル: api/
```

---

## 生成されるContract-First実装

```yaml
# api/openapi.yaml — API仕様（コードより先に書く）
openapi: "3.1.0"
info:
  title: MyApp API
  version: "1.0.0"
  description: |
    MyAppのREST API仕様。
    変更時はCHANGELOG.mdに記録すること。

servers:
  - url: https://api.myapp.com/v1
  - url: http://localhost:3000/api/v1

paths:
  /orders:
    post:
      operationId: createOrder
      summary: 注文を作成する
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: 注文作成成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /orders/{orderId}:
    get:
      operationId: getOrder
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required: [items]
      properties:
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItem'
        couponCode:
          type: string
          pattern: '^[A-Z0-9]{8}$'
          nullable: true

    OrderItem:
      type: object
      required: [productId, quantity]
      properties:
        productId:
          type: string
          format: uuid
        quantity:
          type: integer
          minimum: 1
          maximum: 100

    Order:
      type: object
      required: [id, status, totalCents, createdAt]
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: [pending, confirmed, shipped, delivered, cancelled]
        totalCents:
          type: integer
          minimum: 0
        createdAt:
          type: string
          format: date-time

  responses:
    BadRequest:
      description: バリデーションエラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    NotFound:
      description: リソースが見つかりません
    Unauthorized:
      description: 認証が必要です
```

```typescript
// scripts/generate-types.ts — TypeScript型の自動生成

// package.json scripts:
// "generate": "openapi-typescript api/openapi.yaml -o src/generated/api.ts"

// 生成される型の使用例
import type { paths, components } from '../generated/api';

type CreateOrderRequest = components['schemas']['CreateOrderRequest'];
type Order = components['schemas']['Order'];

// APIハンドラーで型安全に使用
type CreateOrderHandler = (
  req: Request<{}, {}, CreateOrderRequest>,
  res: Response<Order>
) => Promise<void>;
```

```typescript
// src/middleware/validateRequest.ts — OpenAPIスキーマからZodバリデーション

import { z } from 'zod';

// OpenAPIスキーマに対応するZodスキーマ（手動で同期）
// または openapi-zod-client で自動生成
export const CreateOrderRequestSchema = z.object({
  items: z.array(
    z.object({
      productId: z.string().uuid(),
      quantity: z.number().int().min(1).max(100),
    })
  ).min(1),
  couponCode: z.string().regex(/^[A-Z0-9]{8}$/).nullable().optional(),
});

// バリデーションミドルウェア
export function validate<T>(schema: z.ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.issues.map(issue => ({
          path: issue.path.join('.'),
          message: issue.message,
        })),
      });
    }

    req.body = result.data; // パース済みデータで上書き
    next();
  };
}

// 使用例
router.post(
  '/orders',
  requireAuth,
  validate(CreateOrderRequestSchema),
  createOrderHandler
);
```

```bash
# Mockサーバー（Prism）— フロントエンドが先行開発に使用
# npm install -g @stoplight/prism-cli

# 起動
prism mock api/openapi.yaml --port 4010

# フロントエンドはlocalhost:4010を向けて開発可能
# レスポンスはOpenAPI仕様のexampleから自動生成
```

```yaml
# .github/workflows/api-validation.yml
# 実装と仕様の整合性をCIで確認

- name: Validate API implementation against spec
  run: |
    # サーバーを起動してOpenAPIスペックと比較
    npm start &
    sleep 5
    npx openapi-diff api/openapi.yaml http://localhost:3000/api-docs
    # または Dreddで仕様に沿ったHTTPテスト実行
    npx dredd api/openapi.yaml http://localhost:3000/api/v1
```

---

## まとめ

Claude CodeでAPI Contract-Firstを設計する：

1. **CLAUDE.md** にOpenAPI先書き・TypeScript自動生成・仕様と実装の乖離禁止を明記
2. **openapi-typescript** でOpenAPIからTypeScriptの型を自動生成（手書き不要）
3. **Prism Mockサーバー** でバックエンド完成前にフロントエンドが開発開始
4. **CIでDredd** を実行して仕様通りのレスポンスが返っているかを自動検証

---

*API Contract-First設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
