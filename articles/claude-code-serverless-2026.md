---
title: "Claude CodeでServerlessを設計する：AWS Lambda・コールドスタート・Vercel Edge"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "serverless"]
published: true
---

## はじめに

Lambdaのコールドスタートでレスポンスが遅い、DBコネクションが毎回作られてPostgreSQLが限界になる——サーバーレスには独特の設計パターンが必要だ。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにServerlessルールを書く

```markdown
## Serverless設計ルール

### コールドスタート対策
- Lambdaハンドラー外で初期化（DB接続、設定読み込み）
- Provisioned Concurrencyで重要エンドポイントを常時warm
- バンドルサイズ最小化（不要な依存を含めない）

### DB接続
- 接続プールは使わない（Lambda = 1リクエスト = 1インスタンス）
- RDS Proxy経由でPostgreSQLに接続（接続数上限対策）
- Lambda実行環境はリユースされるが、新しいインスタンスも立ち上がる

### 関数設計
- 1関数 = 1責務（ルーティングを詰め込まない）
- タイムアウトは必要最小限（デフォルト3秒は短すぎる場合がある）
- 冪等性を考慮（リトライされることを前提に設計）

### Vercel Edge Functions
- Node.js APIは使えない（Edge Runtime）
- レスポンスは軽量に（50ms以内を目標）
- JWTの検証等、純粋な計算に特化
```

---

## Lambda関数の生成

```
AWS Lambdaでユーザー登録APIを設計してください。

要件：
- コールドスタート対策（ハンドラー外で初期化）
- RDS Proxy経由でPostgreSQL接続
- Zodでバリデーション
- エラーを適切なHTTPステータスコードで返す

生成ファイル: functions/createUser/handler.ts
```

---

## 生成されるLambda関数

```typescript
// functions/createUser/handler.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { PrismaClient } from '@prisma/client';
import { z } from 'zod';

// ハンドラー外で初期化（Lambdaインスタンスリユース時はスキップ）
const prisma = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_URL },
    // RDS Proxyを使う場合: ?connection_limit=1&pool_timeout=0
  },
});

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  password: z.string().min(8),
});

// Lambda handler
export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const requestId = event.requestContext?.requestId ?? crypto.randomUUID();

  try {
    const body = JSON.parse(event.body ?? '{}');
    const result = createUserSchema.safeParse(body);

    if (!result.success) {
      return {
        statusCode: 422,
        headers: { 'Content-Type': 'application/json', 'X-Request-ID': requestId },
        body: JSON.stringify({
          errors: result.error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        }),
      };
    }

    const { email, name, password } = result.data;

    // 重複チェック
    const existing = await prisma.user.findUnique({ where: { email } });
    if (existing) {
      return {
        statusCode: 409,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Email already registered' }),
      };
    }

    const hashedPassword = await bcrypt.hash(password, 12);
    const user = await prisma.user.create({
      data: { email, name, passwordHash: hashedPassword },
      select: { id: true, email: true, name: true, createdAt: true },
    });

    return {
      statusCode: 201,
      headers: { 'Content-Type': 'application/json', 'X-Request-ID': requestId },
      body: JSON.stringify(user),
    };

  } catch (err) {
    console.error({ err, requestId }, 'Unexpected error');
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ error: 'Internal server error' }),
    };
  }
};
```

---

## Vercel Edge Functions（JWT検証）

```typescript
// middleware.ts (Vercel Edge Runtime)
import { NextRequest, NextResponse } from 'next/server';
import { jwtVerify } from 'jose'; // Node.js APIを使わないJWT実装

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('access_token')?.value;

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  try {
    const { payload } = await jwtVerify(token, JWT_SECRET);

    // 検証済みのユーザー情報をヘッダーに付与してバックエンドに渡す
    const response = NextResponse.next();
    response.headers.set('X-User-Id', payload.userId as string);
    response.headers.set('X-Tenant-Id', payload.tenantId as string);
    return response;

  } catch (err) {
    // JWTが無効 → ログインへリダイレクト
    return NextResponse.redirect(new URL('/login', request.url));
  }
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/protected/:path*'],
};
```

---

## まとめ

Claude CodeでServerlessを設計する：

1. **CLAUDE.md** にコールドスタート対策・DB接続方針・Edgeの制約を明記
2. **ハンドラー外で初期化** してLambdaインスタンスリユース時の初期化コストを削減
3. **RDS Proxy** でLambdaからのDB接続数爆発を防ぐ
4. **Vercel Edge** ではNode.js APIを使わない（jose等のEdge対応ライブラリを使う）

---

*サーバーレス設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
