---
title: "Claude Codeで相関IDとリクエストトレーシングを設計する：分散ログ追跡・AsyncLocalStorage"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "observability", "redis"]
published: true
published_at: "2026-03-16 16:00"
---

## はじめに

「エラーログがあるがどのリクエストから来たか追えない」「マイクロサービス間をまたいだ処理が追跡できない」——相関ID（Correlation ID）とAsyncLocalStorageでリクエストを追跡し、全ログを関連付ける設計をClaude Codeに生成させる。

---

## CLAUDE.mdに相関IDトレーシング設計ルールを書く

```markdown
## 相関IDトレーシング設計ルール

### 相関IDの生成・伝播
- リクエスト受信時にULIDで相関IDを生成（なければ）
- X-Correlation-IdヘッダーとX-Request-Idヘッダーを引き継ぐ
- 全ログに相関IDを自動付与（AsyncLocalStorageで伝播）
- 下流サービスへのリクエストにも相関IDをヘッダーで転送

### AsyncLocalStorage活用
- リクエストコンテキストをAsyncLocalStorageに保存
- Promiseチェーン・async/awaitを通じて自動伝播
- DB/Redis/HTTP各クライアントに相関IDを自動付与

### ログ構造
- 全ログに: requestId, correlationId, userId, tenantId, service, version
- 外部サービス呼び出し: method, url, status, durationMs
```

---

## 相関IDトレーシングの生成

```
相関IDとリクエストトレーシングシステムを設計してください。

要件：
- AsyncLocalStorageでコンテキスト伝播
- 全ログへの自動ID付与
- 下流サービスへのヘッダー伝播
- HTTPクライアントラッパー

生成ファイル: src/observability/tracing/
```

---

## 生成される相関IDトレーシング実装

```typescript
// src/observability/tracing/context.ts — リクエストコンテキスト

import { AsyncLocalStorage } from 'async_hooks';
import { ulid } from 'ulid';

export interface RequestContext {
  requestId: string;       // このリクエスト固有のID
  correlationId: string;   // 上流から引き継いだ相関ID（トレース全体のID）
  userId?: string;
  tenantId?: string;
  startedAt: number;
  service: string;
}

const storage = new AsyncLocalStorage<RequestContext>();

export const requestContext = {
  get(): RequestContext | undefined {
    return storage.getStore();
  },

  getOrThrow(): RequestContext {
    const ctx = storage.getStore();
    if (!ctx) throw new Error('No request context found. Did you set up the middleware?');
    return ctx;
  },

  run<T>(context: RequestContext, fn: () => T): T {
    return storage.run(context, fn);
  },
};
```

```typescript
// src/observability/tracing/middleware.ts — Expressミドルウェア

export function requestContextMiddleware(serviceName: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const requestId = ulid();
    // X-Correlation-Idがあれば引き継ぐ、なければ新規生成
    const correlationId = req.headers['x-correlation-id'] as string ?? requestId;

    const context: RequestContext = {
      requestId,
      correlationId,
      userId: undefined, // authミドルウェアが後で設定
      tenantId: undefined,
      startedAt: Date.now(),
      service: serviceName,
    };

    // レスポンスヘッダーに付与（クライアントがトレースできるように）
    res.setHeader('X-Request-Id', requestId);
    res.setHeader('X-Correlation-Id', correlationId);

    // AsyncLocalStorageにセット（全ての非同期処理に自動伝播）
    requestContext.run(context, next);
  };
}

// 認証後にユーザー情報をコンテキストに追加
export function attachUserToContext(req: Request, _res: Response, next: NextFunction) {
  if (req.userId) {
    const ctx = requestContext.get();
    if (ctx) {
      (ctx as any).userId = req.userId;
      (ctx as any).tenantId = req.user?.tenantId;
    }
  }
  next();
}
```

```typescript
// src/observability/tracing/tracingLogger.ts — コンテキスト付きロガー

import pino from 'pino';

const baseLogger = pino({ level: process.env.LOG_LEVEL ?? 'info' });

// 常にリクエストコンテキストを含むロガー
export const logger = {
  info: (obj: object | string, msg?: string) => {
    const ctx = requestContext.get();
    const context = ctx ? { requestId: ctx.requestId, correlationId: ctx.correlationId, userId: ctx.userId } : {};
    typeof obj === 'string' ? baseLogger.info(context, obj) : baseLogger.info({ ...context, ...obj }, msg);
  },
  warn: (obj: object | string, msg?: string) => {
    const ctx = requestContext.get();
    const context = ctx ? { requestId: ctx.requestId, correlationId: ctx.correlationId } : {};
    typeof obj === 'string' ? baseLogger.warn(context, obj) : baseLogger.warn({ ...context, ...obj }, msg);
  },
  error: (obj: object | string, msg?: string) => {
    const ctx = requestContext.get();
    const context = ctx ? { requestId: ctx.requestId, correlationId: ctx.correlationId, userId: ctx.userId } : {};
    typeof obj === 'string' ? baseLogger.error(context, obj) : baseLogger.error({ ...context, ...obj }, msg);
  },
};

// ログ例: { requestId: '01HRZ...', correlationId: '01HRY...', userId: 'u123', message: 'Order created' }
```

```typescript
// src/observability/tracing/tracingHttpClient.ts — 相関ID自動転送HTTPクライアント

export async function tracedFetch(url: string, options: RequestInit = {}): Promise<Response> {
  const ctx = requestContext.get();

  const headers = new Headers(options.headers);

  // 相関IDを下流サービスへ転送（分散トレーシングの継続）
  if (ctx) {
    headers.set('X-Correlation-Id', ctx.correlationId);
    headers.set('X-Request-Id', ulid()); // 新しいリクエストIDは再生成
    headers.set('X-Upstream-Service', ctx.service);
  }

  const startMs = Date.now();
  let response: Response;

  try {
    response = await fetch(url, { ...options, headers });
  } catch (error) {
    logger.error({ url, error: (error as Error).message, durationMs: Date.now() - startMs }, 'HTTP request failed');
    throw error;
  }

  logger.info({
    method: (options.method ?? 'GET').toUpperCase(),
    url,
    status: response.status,
    durationMs: Date.now() - startMs,
  }, 'Outbound HTTP request');

  return response;
}

// Prismaのクエリログ拡張（クエリに相関IDを付与）
export function createTracingPrismaClient(): PrismaClient {
  return new PrismaClient({
    log: [{ emit: 'event', level: 'query' }],
  }).$extends({
    query: {
      async $allOperations({ operation, model, args, query }) {
        const startMs = Date.now();
        const result = await query(args);
        const durationMs = Date.now() - startMs;

        if (durationMs > 100) { // 100ms以上のクエリをログ
          logger.warn({ model, operation, durationMs }, 'Slow query detected');
        }

        return result;
      },
    },
  });
}

// 使用例: アプリケーション全体での設定
app.use(requestContextMiddleware('api-service'));
app.use(requireAuth);
app.use(attachUserToContext);

// 全APIルートで自動的に相関IDが全ログに付与される
router.post('/api/orders', requireAuth, async (req, res) => {
  logger.info({ userId: req.userId }, 'Creating order'); // 自動で requestId/correlationId が付く
  const order = await orderService.create(req.userId, req.body.items);
  logger.info({ orderId: order.id }, 'Order created');
  res.status(201).json(order);
});
```

---

## まとめ

Claude Codeで相関IDとリクエストトレーシングを設計する：

1. **CLAUDE.md** にULIDで相関ID生成・X-Correlation-Idヘッダー引き継ぎ・全ログへの自動付与・下流サービスへのヘッダー転送を明記
2. **AsyncLocalStorage** でPromiseチェーン全体にコンテキストを自動伝播——`req`オブジェクトを引き回すことなく、どこからでも相関IDを取得可能
3. **tracedFetch** で外部HTTP呼び出しに相関IDを自動付与——マイクロサービス間をまたいで同じcorrelationIdで追跡できる
4. **Prismaクエリ拡張** でスロークエリ（100ms超）を相関ID付きでログ——ボトルネックのリクエストを特定するのが容易

---

*観測可能性のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
