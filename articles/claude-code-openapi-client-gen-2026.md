---
title: "Claude CodeでOpenAPIクライアント自動生成を設計する：型安全SDK・モックサーバー・CI同期"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "openapi", "devops"]
published: true
published_at: "2026-03-14 05:00"
---

## はじめに

APIの変更をフロントエンドチームに手動で伝えるのは終わり——OpenAPI specからTypeScript型とAPIクライアントを自動生成し、spec変更をCIで検知してPRコメントで差分を通知する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにOpenAPIクライアント生成設計ルールを書く

```markdown
## OpenAPIクライアント自動生成設計ルール

### Spec管理
- openapi.yaml はルートに配置、単一ソースオブジェクト
- APIの変更はコード変更前にSpecを更新（Spec First）
- バージョニング: info.version を semver で管理

### クライアント生成
- openapi-typescript: TypeScript型のみ生成（軽量）
- openapi-fetch: 型付きfetchクライアント生成
- 生成物はsrc/generated/に配置（コミットする）

### CI検知
- specが変更されたPRでクライアントを再生成
- 差分があれば → PR失敗（生成物の更新を強制）
- Prism mock server でE2Eテストのバックエンド代替
```

---

## OpenAPIクライアント生成の生成

```
OpenAPI specからTypeScript SDKを自動生成するパイプラインを設計してください。

要件：
- openapi-typescriptによる型生成
- openapi-fetchクライアント
- Prismモックサーバー
- CI生成・差分検知
- Breaking change検出

生成ファイル: openapi.yaml, src/generated/, .github/workflows/
```

---

## 生成されるOpenAPIクライアント生成パイプライン

```yaml
# openapi.yaml — API仕様（単一ソース）

openapi: "3.1.0"
info:
  title: MyApp API
  version: "2.1.0"

paths:
  /users/{id}:
    get:
      operationId: getUserById
      summary: ユーザー詳細取得
      tags: [users]
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        "200":
          description: Success
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
        "404":
          description: Not found
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /orders:
    post:
      operationId: createOrder
      summary: 注文作成
      tags: [orders]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateOrderRequest' }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        "400":
          description: Validation error
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ValidationError' }

components:
  schemas:
    User:
      type: object
      required: [id, email, name, createdAt]
      properties:
        id:    { type: string, format: uuid }
        email: { type: string, format: email }
        name:  { type: string, minLength: 1, maxLength: 100 }
        role:  { type: string, enum: [user, admin] }
        createdAt: { type: string, format: date-time }

    CreateOrderRequest:
      type: object
      required: [items]
      properties:
        items:
          type: array
          minItems: 1
          items:
            type: object
            required: [productId, quantity]
            properties:
              productId: { type: string, format: uuid }
              quantity:  { type: integer, minimum: 1, maximum: 100 }

    Error:
      type: object
      required: [code, message]
      properties:
        code:    { type: string }
        message: { type: string }
```

```typescript
// scripts/generate-client.ts — クライアント生成スクリプト

import { execSync } from 'child_process';
import { writeFileSync } from 'fs';

// 1. TypeScript型定義を生成
execSync('npx openapi-typescript openapi.yaml -o src/generated/api.d.ts', { stdio: 'inherit' });

// 2. 型付きfetchクライアントを生成（openapi-fetch用のラッパー）
const clientCode = `
// This file is auto-generated. Do not edit manually.
// Run: npm run generate:client

import createClient from 'openapi-fetch';
import type { paths } from './api.d.ts';

export const apiClient = createClient<paths>({
  baseUrl: process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3000',
  headers: {
    'Content-Type': 'application/json',
  },
});

// AuthorizationヘッダーをInterceptorで追加
apiClient.use({
  onRequest({ request }) {
    const token = getAccessToken(); // メモリから取得
    if (token) request.headers.set('Authorization', \`Bearer \${token}\`);
    return request;
  },
});
`;
writeFileSync('src/generated/client.ts', clientCode);
console.log('Client generated successfully');
```

```typescript
// src/generated/client.ts の使用例（自動補完が効く）

import { apiClient } from './generated/client';

// 完全型安全: パス・メソッド・レスポンスが全て型推論
const { data, error } = await apiClient.GET('/users/{id}', {
  params: { path: { id: userId } }, // UUID型チェック
});

if (data) {
  console.log(data.email); // string として推論
  console.log(data.role);  // 'user' | 'admin' として推論
}

const { data: order, error: orderError } = await apiClient.POST('/orders', {
  body: {
    items: [{ productId: 'uuid-here', quantity: 2 }], // Zod-likeな型チェック
  },
});
```

```yaml
# .github/workflows/api-client-sync.yml

name: API Client Sync

on:
  pull_request:
    paths:
      - 'openapi.yaml'

jobs:
  generate-and-check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }

      - run: npm ci

      - name: Generate client from spec
        run: npm run generate:client

      - name: Check for uncommitted changes
        id: diff
        run: |
          git diff --name-only src/generated/
          echo "has_changes=$(git diff --name-only src/generated/ | wc -l)" >> $GITHUB_OUTPUT

      - name: Fail if generated files are outdated
        if: steps.diff.outputs.has_changes != '0'
        run: |
          echo "Generated files are outdated! Run 'npm run generate:client' and commit the changes."
          exit 1

  breaking-change-check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Get base branch spec
        run: git show origin/main:openapi.yaml > /tmp/openapi-base.yaml

      - name: Check for breaking changes
        uses: oasdiff/oasdiff-action@main
        with:
          base: /tmp/openapi-base.yaml
          revision: openapi.yaml
          fail-on-diff: breaking

      - name: Comment breaking changes on PR
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '⚠️ **Breaking API Changes Detected!**\n\nThis PR contains breaking API changes. Please bump the major version in `openapi.yaml` and notify API consumers.',
            });
```

---

## まとめ

Claude CodeでOpenAPIクライアント自動生成を設計する：

1. **CLAUDE.md** にSpec First開発・生成物をコミット・spec変更でCI再生成・差分あればPR失敗を明記
2. **openapi-fetch** で`paths`型からエンドポイント・パラメータ・レスポンスが全て型推論——手書きAPI呼び出しを排除
3. **oasdiff** でbreaking change（フィールド削除・型変更）を自動検出しPRコメントで通知
4. **Prism mock server** でOpenAPI specから自動的にモックAPIを起動——フロントエンドが独立開発可能

---

*API設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
