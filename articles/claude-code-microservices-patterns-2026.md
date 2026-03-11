---
title: "Claude Codeでマイクロサービス間通信を設計する：Circuit Breaker・サービスディスカバリー"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "microservices", "architecture"]
published: true
---

## はじめに

マイクロサービス間のHTTP呼び出しで、1つのサービスが落ちると連鎖的に全体が止まる（Cascade Failure）。Circuit BreakerとRetryで障害を局所化する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにマイクロサービス通信ルールを書く

```markdown
## マイクロサービス間通信ルール

### 必須パターン
- Circuit Breaker: 連続失敗後に呼び出しを一時遮断（Cascade Failure防止）
- Retry with Exponential Backoff: 一時的な失敗のみリトライ
- Timeout: 全サービス呼び出しに3秒タイムアウト
- Bulkhead: サービスごとに接続プールを分離

### サービス間認証
- JWT（内部用: 短命・service-to-service専用）
- mTLS（本番環境推奨）
- APIキー（シンプルだがローテーション運用必要）

### エラー分類
- 4xx（クライアントエラー）: リトライしない
- 5xx（サーバーエラー）: 指数バックオフでリトライ
- タイムアウト/接続エラー: リトライ（Circuit Breaker経由）

### Observability
- 全サービス呼び出しにtraceId を伝播（OpenTelemetry）
- Circuit Breaker状態変化をメトリクスに記録
- サービス依存グラフを可視化
```

---

## Circuit Breaker実装の生成

```
Circuit BreakerとRetryを組み合わせたHTTPクライアントを設計してください。

要件：
- Circuit Breaker（CLOSED/OPEN/HALF-OPEN状態管理）
- 指数バックオフリトライ（4xxはリトライしない）
- タイムアウト設定
- ヘルスチェック（HALF-OPEN状態でテスト呼び出し）

生成ファイル: src/clients/resilientClient.ts
```

---

## 生成されるCircuit Breaker実装

```typescript
// src/clients/circuitBreaker.ts

type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerOptions {
  failureThreshold: number;    // 連続失敗数でOPENに（デフォルト5）
  recoveryTimeout: number;     // OPENからHALF_OPENに移行する時間（ms）
  halfOpenSuccessThreshold: number; // HALF_OPENから CLOSEDに移行する成功数
}

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime = 0;
  private halfOpenSuccessCount = 0;

  constructor(
    private readonly name: string,
    private readonly options: CircuitBreakerOptions = {
      failureThreshold: 5,
      recoveryTimeout: 30_000,
      halfOpenSuccessThreshold: 3,
    }
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      // タイムアウト後はHALF_OPENに移行
      if (Date.now() - this.lastFailureTime > this.options.recoveryTimeout) {
        this.state = 'HALF_OPEN';
        logger.info({ service: this.name }, 'Circuit breaker: OPEN → HALF_OPEN');
      } else {
        throw new ServiceUnavailableError(`Circuit breaker OPEN for ${this.name}`);
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure(err);
      throw err;
    }
  }

  private onSuccess(): void {
    if (this.state === 'HALF_OPEN') {
      this.halfOpenSuccessCount++;
      if (this.halfOpenSuccessCount >= this.options.halfOpenSuccessThreshold) {
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.halfOpenSuccessCount = 0;
        logger.info({ service: this.name }, 'Circuit breaker: HALF_OPEN → CLOSED');
      }
    } else {
      this.failureCount = 0; // 成功したらカウントリセット
    }
  }

  private onFailure(err: unknown): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.state === 'HALF_OPEN') {
      this.state = 'OPEN';
      this.halfOpenSuccessCount = 0;
      logger.warn({ service: this.name }, 'Circuit breaker: HALF_OPEN → OPEN');
    } else if (this.failureCount >= this.options.failureThreshold) {
      this.state = 'OPEN';
      logger.error({
        service: this.name,
        failureCount: this.failureCount,
        err,
      }, 'Circuit breaker: CLOSED → OPEN');
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

```typescript
// src/clients/resilientClient.ts
import { CircuitBreaker } from './circuitBreaker';

interface RetryOptions {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = { maxAttempts: 3, baseDelayMs: 100, maxDelayMs: 5000 }
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err as Error;

      // 4xxエラーはリトライしない（クライアント側の問題）
      if (err instanceof HttpError && err.status >= 400 && err.status < 500) {
        throw err;
      }

      if (attempt < options.maxAttempts - 1) {
        const delay = Math.min(
          options.baseDelayMs * Math.pow(2, attempt),
          options.maxDelayMs
        );
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError!;
}

// Circuit Breaker + Retry を組み合わせたHTTPクライアント
export class ResilientClient {
  private circuitBreakers = new Map<string, CircuitBreaker>();

  private getCircuitBreaker(serviceName: string): CircuitBreaker {
    if (!this.circuitBreakers.has(serviceName)) {
      this.circuitBreakers.set(serviceName, new CircuitBreaker(serviceName));
    }
    return this.circuitBreakers.get(serviceName)!;
  }

  async get<T>(
    serviceName: string,
    url: string,
    options?: RequestInit,
    timeoutMs = 3000
  ): Promise<T> {
    const cb = this.getCircuitBreaker(serviceName);

    return cb.execute(() =>
      withRetry(async () => {
        const controller = new AbortController();
        const timeout = setTimeout(() => controller.abort(), timeoutMs);

        try {
          const res = await fetch(url, {
            ...options,
            signal: controller.signal,
            headers: {
              ...options?.headers,
              'X-Trace-ID': getCurrentTraceId(), // OpenTelemetry traceId伝播
            },
          });

          if (!res.ok) {
            throw new HttpError(res.status, await res.text());
          }

          return res.json() as Promise<T>;
        } finally {
          clearTimeout(timeout);
        }
      })
    );
  }
}

export const serviceClient = new ResilientClient();
```

---

## サービスクライアントの使用例

```typescript
// src/services/orderService.ts
class OrderServiceClient {
  private baseUrl = process.env.ORDER_SERVICE_URL!;

  async getOrder(orderId: string): Promise<Order> {
    return serviceClient.get<Order>(
      'order-service', // Circuit Breakerの識別名
      `${this.baseUrl}/orders/${orderId}`
    );
  }

  async createOrder(data: CreateOrderInput): Promise<Order> {
    return serviceClient.post<Order>('order-service', `${this.baseUrl}/orders`, data);
  }
}

// フォールバック付きの使用
async function getOrderWithFallback(orderId: string): Promise<Order | null> {
  try {
    return await orderClient.getOrder(orderId);
  } catch (err) {
    if (err instanceof ServiceUnavailableError) {
      // Circuit Breaker OPENの場合: キャッシュから返す
      logger.warn({ orderId }, 'Order service unavailable, using cache');
      return redis.get(`order:${orderId}`).then(cached =>
        cached ? JSON.parse(cached) : null
      );
    }
    throw err;
  }
}
```

---

## まとめ

Claude Codeでマイクロサービス間通信を設計する：

1. **CLAUDE.md** にCircuit Breaker必須・4xxはリトライしない・タイムアウト3秒を明記
2. **Circuit Breaker** で連続失敗後に呼び出しを遮断（Cascade Failure防止）
3. **HALF_OPEN状態** でテスト呼び出しを行い、段階的に回復
4. **フォールバック** でCircuit Breaker OPENの場合もキャッシュから応答

---

*マイクロサービス設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
