---
title: "Claude CodeでEdge Functionsを設計する：Cloudflare Workers・Vercel Edge・グローバル低レイテンシ"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "typescript", "cloudflare", "edge", "performance"]
published: true
published_at: "2026-03-12 14:00"
---

## はじめに

サーバーレスでも低レイテンシが必要なユースケース——Cloudflare WorkersとVercel Edge Runtimeでエッジに処理を分散し、世界中のユーザーに10ms以下のレスポンスを提供する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにEdge Function設計ルールを書く

```markdown
## Edge Function設計ルール

### 適用範囲
- A/Bテスト（ユーザーをエッジで振り分け）
- 認証チェック（JWTを検証してオリジンへ転送）
- レート制限（DDoS保護）
- ジオルーティング（国別コンテンツ）
- 軽量な変換（ヘッダー追加・レスポンス変換）

### 制約事項
- 実行時間: 最大50ms（CPU時間）
- メモリ: 128MB以下
- ファイルシステム: なし（KV/D1/R2で代替）
- Node.jsモジュール: 使用不可（Web APIのみ）

### データストア
- セッション・カウンター: Cloudflare KV
- 構造化データ: Cloudflare D1（SQLite）
- ファイル: Cloudflare R2
```

---

## Edge Function実装の生成

```
Cloudflare Workers Edge Functionを設計してください。

要件：
- JWT認証ミドルウェア
- レート制限（KV使用）
- A/Bテスト（Cookie-based）
- ジオルーティング
- Cache API活用

生成ファイル: workers/
```

---

## 生成されるEdge Function実装

```typescript
// workers/auth-middleware.ts — JWT検証エッジミドルウェア

interface Env {
  JWT_SECRET: string;
  RATE_LIMIT_KV: KVNamespace;
  ORIGIN_URL: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);

    // ヘルスチェックはスルー
    if (url.pathname === '/health') {
      return new Response('OK', { status: 200 });
    }

    // レート制限チェック（エッジで実施 → オリジンに到達させない）
    const rateLimitResult = await checkRateLimit(request, env);
    if (!rateLimitResult.allowed) {
      return new Response('Too Many Requests', {
        status: 429,
        headers: {
          'Retry-After': String(rateLimitResult.retryAfterSeconds),
          'X-RateLimit-Limit': String(rateLimitResult.limit),
          'X-RateLimit-Remaining': '0',
        },
      });
    }

    // JWT検証（エッジで実施 → オリジンの認証負荷を削減）
    const authHeader = request.headers.get('Authorization');
    if (authHeader?.startsWith('Bearer ')) {
      const token = authHeader.slice(7);
      const payload = await verifyJWT(token, env.JWT_SECRET);

      if (!payload) {
        return new Response('Unauthorized', { status: 401 });
      }

      // 検証済みユーザー情報をオリジンに転送
      const modifiedRequest = new Request(request, {
        headers: new Headers({
          ...Object.fromEntries(request.headers.entries()),
          'X-User-Id': payload.sub as string,
          'X-User-Role': payload.role as string,
          'X-CF-Worker': 'verified', // オリジンが再検証不要なシグナル
        }),
      });

      return fetch(`${env.ORIGIN_URL}${url.pathname}${url.search}`, modifiedRequest);
    }

    // 未認証リクエストはそのままオリジンへ（オリジンが401を返す）
    return fetch(`${env.ORIGIN_URL}${url.pathname}${url.search}`, request);
  },
};

// レート制限（Cloudflare KV使用）
async function checkRateLimit(
  request: Request,
  env: Env
): Promise<{ allowed: boolean; limit: number; retryAfterSeconds?: number }> {
  const ip = request.headers.get('CF-Connecting-IP') ?? 'unknown';
  const windowKey = `ratelimit:${ip}:${Math.floor(Date.now() / 60_000)}`; // 1分ウィンドウ

  const currentStr = await env.RATE_LIMIT_KV.get(windowKey);
  const current = parseInt(currentStr ?? '0');
  const limit = 100; // 1分間に100リクエスト

  if (current >= limit) {
    return { allowed: false, limit, retryAfterSeconds: 60 };
  }

  // カウントを増やす（TTL: 90秒で自動削除）
  await env.RATE_LIMIT_KV.put(windowKey, String(current + 1), { expirationTtl: 90 });

  return { allowed: true, limit };
}

// JWT検証（Web Crypto API使用: Node.js不要）
async function verifyJWT(token: string, secret: string): Promise<Record<string, unknown> | null> {
  try {
    const [headerB64, payloadB64, signatureB64] = token.split('.');
    const encoder = new TextEncoder();

    const key = await crypto.subtle.importKey(
      'raw',
      encoder.encode(secret),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['verify']
    );

    const data = encoder.encode(`${headerB64}.${payloadB64}`);
    const signature = Uint8Array.from(atob(signatureB64.replace(/-/g, '+').replace(/_/g, '/')), c => c.charCodeAt(0));

    const valid = await crypto.subtle.verify('HMAC', key, signature, data);
    if (!valid) return null;

    const payload = JSON.parse(atob(payloadB64));
    if (payload.exp && payload.exp < Date.now() / 1000) return null;

    return payload;
  } catch {
    return null;
  }
}
```

```typescript
// workers/ab-test.ts — A/Bテスト（エッジで振り分け）

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // A/Bテスト対象パスのみ
    if (!url.pathname.startsWith('/checkout')) {
      return fetch(request);
    }

    // 既存のVariant Cookieを確認
    const cookies = parseCookies(request.headers.get('Cookie') ?? '');
    let variant = cookies['ab_checkout'];

    if (!variant) {
      // 新規ユーザー: Sticky割り当て（ユーザーIDかIPで決定論的に振り分け）
      const userId = request.headers.get('X-User-Id');
      const identifier = userId ?? request.headers.get('CF-Connecting-IP') ?? 'anon';
      const hash = await hashString(identifier + 'checkout_v2');
      variant = hash[0] < 128 ? 'control' : 'treatment'; // 50/50
    }

    // Variantに応じてリクエストを転送
    const targetUrl = variant === 'treatment'
      ? `${env.ORIGIN_URL}/checkout-v2${url.search}`
      : `${env.ORIGIN_URL}/checkout${url.search}`;

    const response = await fetch(targetUrl, request);

    // レスポンスにVariant Cookieを付与
    const modifiedResponse = new Response(response.body, response);
    if (!cookies['ab_checkout']) {
      modifiedResponse.headers.append(
        'Set-Cookie',
        `ab_checkout=${variant}; Path=/; Max-Age=604800; SameSite=Lax`
      );
    }
    modifiedResponse.headers.set('X-AB-Variant', variant);

    return modifiedResponse;
  },
};

async function hashString(str: string): Promise<Uint8Array> {
  const data = new TextEncoder().encode(str);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  return new Uint8Array(hashBuffer);
}
```

---

## Cache APIでAPIレスポンスをエッジキャッシュ

```typescript
// workers/api-cache.ts — エッジキャッシュ

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);
    const cache = caches.default;

    // GETリクエストのみキャッシュ
    if (request.method !== 'GET') {
      return fetch(request);
    }

    // キャッシュを確認
    const cachedResponse = await cache.match(request);
    if (cachedResponse) {
      return new Response(cachedResponse.body, {
        ...cachedResponse,
        headers: new Headers({
          ...Object.fromEntries(cachedResponse.headers.entries()),
          'X-Cache': 'HIT',
        }),
      });
    }

    // オリジンから取得
    const response = await fetch(request);

    // Cache-Controlヘッダーがあればエッジにキャッシュ
    if (response.headers.get('Cache-Control')?.includes('s-maxage')) {
      ctx.waitUntil(cache.put(request, response.clone()));
    }

    return new Response(response.body, {
      ...response,
      headers: new Headers({
        ...Object.fromEntries(response.headers.entries()),
        'X-Cache': 'MISS',
      }),
    });
  },
};
```

---

## まとめ

Claude CodeでEdge Functionsを設計する：

1. **CLAUDE.md** に適用範囲・50ms制限・Node.js不可・KV/D1/R2データストアを明記
2. **JWT検証をエッジで** 実施してオリジンの認証負荷を削減（Web Crypto API使用）
3. **レート制限をKVで** 実施してDDoSをオリジンに到達させない
4. **A/BテストはCookieで** スティッキーセッションを維持（ユーザーごとに決定論的）

---

*Edge Function設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
