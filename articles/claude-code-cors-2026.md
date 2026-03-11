---
title: "Claude CodeでCORS設定を設計する：オリジン制御とプリフライト最適化"
emoji: "🌐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "cors", "security"]
published: true
---

## はじめに

CORSの設定ミスはセキュリティホールになる——`Access-Control-Allow-Origin: *` のまま本番環境に出すと、任意サイトからAPIを叩かれる。Claude Codeに安全なCORS設定を生成させる。

---

## CLAUDE.mdにCORSルールを書く

```markdown
## CORS設定ルール

### セキュリティ（必須）
- 本番環境で `Access-Control-Allow-Origin: *` 禁止
- 許可オリジンはenv変数から読み込む（ハードコード禁止）
- credentialsを使う場合は特定オリジンのみ許可（*と組み合わせ不可）
- 許可メソッドは必要なものだけ（DELETE等は明示的に許可）

### プリフライト
- CORSミドルウェアは全ルートより前に登録
- OPTIONSリクエストには204を返す（ボディなし）
- プリフライトキャッシュ: max-age=86400（24時間）

### 許可ヘッダー
- リクエスト: Content-Type, Authorization, X-Request-ID
- レスポンス: X-Total-Count, X-Request-ID（カスタムヘッダーはexposeで公開）
```

---

## CORS設定の生成

```
CORS設定を生成してください。

要件：
- 本番/開発で許可オリジンを切り替え
- credentials対応（Cookie認証）
- プリフライトキャッシュ24時間
- カスタムレスポンスヘッダー（X-Total-Count）を公開

保存先: src/middleware/cors.ts
```

---

## 生成されるCORS設定

```typescript
// src/middleware/cors.ts
import cors from 'cors';

const ALLOWED_ORIGINS = process.env.ALLOWED_ORIGINS?.split(',').map(o => o.trim()) ?? [];

export const corsMiddleware = cors({
  origin: (origin, callback) => {
    // Same-origin requests (server-to-server, curl等) はoriginがundefined
    if (!origin) return callback(null, true);

    if (ALLOWED_ORIGINS.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`CORS: origin ${origin} not allowed`));
    }
  },
  credentials: true,                // Cookie/Authorizationヘッダーを許可
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  exposedHeaders: ['X-Total-Count', 'X-Request-ID'], // クライアントから読めるようにする
  maxAge: 86400,                    // プリフライトキャッシュ24時間
});
```

```typescript
// src/app.ts
import express from 'express';
import { corsMiddleware } from './middleware/cors';

const app = express();

// CORSは全ルートより前に登録（必須）
app.use(corsMiddleware);

// OPTIONSプリフライトに204を返す
app.options('*', corsMiddleware, (req, res) => {
  res.status(204).end();
});

app.use(express.json());
// ... 他のルート
```

---

## 動的オリジン検証

```
テナントごとに許可オリジンが異なるマルチテナント向けCORS設定を生成してください。

要件：
- テナントIDをリクエストから特定（サブドメインまたはヘッダー）
- テナントの許可オリジンをDBから取得
- 結果をRedisにキャッシュ（TTL 5分）
```

```typescript
// src/middleware/dynamicCors.ts
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });

async function getAllowedOrigins(tenantId: string): Promise<string[]> {
  const cacheKey = `cors:tenant:${tenantId}`;

  // キャッシュ確認
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // DBから取得
  const tenant = await prisma.tenant.findUnique({
    where: { id: tenantId },
    select: { allowedOrigins: true },
  });

  const origins = tenant?.allowedOrigins ?? [];
  await redis.set(cacheKey, JSON.stringify(origins), { EX: 300 }); // 5分

  return origins;
}

export const dynamicCorsMiddleware = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const tenantId = req.headers['x-tenant-id'] as string;
  const origin = req.headers.origin as string;

  if (!tenantId || !origin) return next();

  const allowedOrigins = await getAllowedOrigins(tenantId);

  if (allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Vary', 'Origin'); // キャッシュに正しいOriginを使わせる
  }

  next();
};
```

---

## 環境変数設定

```bash
# .env.example
# カンマ区切りで複数オリジンを指定
ALLOWED_ORIGINS=https://app.example.com,https://admin.example.com

# 開発環境
# ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

---

## まとめ

Claude CodeでCORS設定を設計する：

1. **CLAUDE.md** に本番`*`禁止・オリジンはenv・credentials注意点を明記
2. **オリジン検証関数** で動的に許可判定（ハードコード禁止）
3. **Varyヘッダー** でCDN/プロキシの誤キャッシュを防ぐ
4. **プリフライトキャッシュ** で余分なOPTIONSリクエストを削減

---

*CORSの設定ミス（`*`許可、credentials漏洩）を自動検出するスキルは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
