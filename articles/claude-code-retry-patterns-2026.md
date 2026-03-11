---
title: "Claude Codeでリトライパターンを設計する：指数バックオフ・ジッター・冪等性・リトライ嵐防止"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-20 12:00"
---

## はじめに

「全クライアントが同時にリトライして外部APIがダウンした」「リトライのたびに副作用が発生してデータが二重登録された」——指数バックオフ・ジッター・冪等性の3要素で安全なリトライを設計をClaude Codeに生成させる。

---

## CLAUDE.mdにリトライパターン設計ルールを書く

```markdown
## リトライパターン設計ルール

### 基本原則
- 冪等な操作のみリトライ（非冪等はリトライしない）
- 指数バックオフ + ランダムジッター（全クライアントの同期リトライを防ぐ）
- 最大リトライ回数とタイムアウトの両方を設定

### バックオフ計算
- 基本: min(base * 2^attempt, maxDelay)
- フルジッター: random(0, calculated_delay)  ← 推奨
- 等分ジッター: calculated_delay/2 + random(0, calculated_delay/2)

### リトライすべきエラー
- 一時的障害: 429(Rate Limit), 503(Unavailable), 504(Gateway Timeout)
- ネットワークエラー: ECONNRESET, ETIMEDOUT, ENOTFOUND
- リトライしない: 400(Bad Request), 401(Unauthorized), 404(Not Found), 422
```

---

## リトライ実装の生成

```
包括的なリトライパターンを設計してください。

要件：
- 指数バックオフ + ジッター
- エラー種別の判定
- 冪等キーの自動付与
- リトライ嵐防止

生成ファイル: src/infrastructure/retry/
```

---

## 生成されるリトライ実装

```typescript
// src/infrastructure/retry/retryPolicy.ts — リトライポリシー

export interface RetryOptions {
  maxAttempts: number;      // 最大試行回数（初回含む）
  baseDelay: number;        // 初回バックオフ（ミリ秒）
  maxDelay: number;         // 最大バックオフ（ミリ秒）
  jitter: 'full' | 'equal' | 'none';  // ジッター戦略
  timeout?: number;         // 全体タイムアウト（ミリ秒）
  retryIf?: (error: Error) => boolean;  // リトライ判定関数
  onRetry?: (attempt: number, error: Error, delay: number) => void;  // フック
}

export const DEFAULT_RETRY_OPTIONS: RetryOptions = {
  maxAttempts: 3,
  baseDelay: 1_000,
  maxDelay: 30_000,
  jitter: 'full',
  retryIf: isRetryableError,
};

// リトライすべきエラーの判定
export function isRetryableError(error: Error): boolean {
  // HTTPステータスコードベース
  if (error instanceof HttpError) {
    return [429, 500, 502, 503, 504].includes(error.statusCode);
  }

  // ネットワークエラー
  const retryableCodes = new Set([
    'ECONNRESET',
    'ECONNREFUSED',
    'ETIMEDOUT',
    'ENOTFOUND',
    'ENETUNREACH',
    'ERR_NETWORK',
  ]);

  if ('code' in error && retryableCodes.has((error as NodeJS.ErrnoException).code ?? '')) {
    return true;
  }

  // エラーメッセージパターン
  const message = error.message.toLowerCase();
  return (
    message.includes('timeout') ||
    message.includes('connection reset') ||
    message.includes('socket hang up') ||
    message.includes('rate limit')
  );
}

// バックオフ計算
export function calculateDelay(attempt: number, options: RetryOptions): number {
  // 指数バックオフ: base * 2^(attempt-1)
  const exponential = Math.min(
    options.baseDelay * Math.pow(2, attempt - 1),
    options.maxDelay
  );

  switch (options.jitter) {
    case 'full':
      // フルジッター: [0, exponential] の一様乱数
      // → 多くのクライアントが分散してリトライ（推奨）
      return Math.random() * exponential;

    case 'equal':
      // 等分ジッター: [exponential/2, exponential] の一様乱数
      // → 最小保証しつつ分散
      return exponential / 2 + Math.random() * (exponential / 2);

    case 'none':
      return exponential;
  }
}

// リトライ実行関数
export async function withRetry<T>(
  fn: (attempt: number) => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...DEFAULT_RETRY_OPTIONS, ...options };
  const startTime = Date.now();

  let lastError: Error = new Error('No attempts made');

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    // 全体タイムアウトチェック
    if (opts.timeout && Date.now() - startTime > opts.timeout) {
      throw new RetryTimeoutError(attempt - 1, opts.timeout, lastError);
    }

    try {
      return await fn(attempt);
    } catch (error) {
      lastError = error as Error;

      const isLast = attempt === opts.maxAttempts;
      const shouldRetry = opts.retryIf ? opts.retryIf(lastError) : isRetryableError(lastError);

      if (isLast || !shouldRetry) {
        throw lastError;
      }

      const delay = calculateDelay(attempt, opts);

      opts.onRetry?.(attempt, lastError, delay);
      logger.warn({ attempt, maxAttempts: opts.maxAttempts, delay, error: lastError.message },
        'Retrying after error');

      await sleep(delay);
    }
  }

  throw lastError;
}
```

```typescript
// src/infrastructure/retry/idempotentHttpClient.ts — 冪等キー自動付与HTTPクライアント

export class IdempotentHttpClient {
  constructor(
    private readonly baseUrl: string,
    private readonly retryOptions: Partial<RetryOptions> = {}
  ) {}

  // 冪等キーを自動生成してリクエストに付与
  async post<TBody, TResponse>(
    path: string,
    body: TBody,
    options: {
      idempotencyKey?: string;  // 指定しない場合は自動生成
      timeout?: number;
    } = {}
  ): Promise<TResponse> {
    const idempotencyKey = options.idempotencyKey ?? ulid();

    return withRetry(
      async (attempt) => {
        const response = await fetch(`${this.baseUrl}${path}`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Idempotency-Key': idempotencyKey,  // 同じキーで再送しても副作用なし
            'X-Attempt-Number': String(attempt),
          },
          body: JSON.stringify(body),
          signal: options.timeout
            ? AbortSignal.timeout(options.timeout)
            : undefined,
        });

        if (!response.ok) {
          const errorBody = await response.json().catch(() => ({}));
          throw new HttpError(response.status, response.statusText, errorBody);
        }

        return response.json() as Promise<TResponse>;
      },
      {
        ...this.retryOptions,
        // 冪等なPOSTのみリトライ可能
        retryIf: (error) => {
          if (error instanceof HttpError && error.statusCode === 422) return false;  // ビジネスエラーはリトライしない
          return isRetryableError(error);
        },
        onRetry: (attempt, error, delay) => {
          logger.info({ path, idempotencyKey, attempt, delay }, 'Retrying HTTP request');
        },
      }
    );
  }

  async get<TResponse>(path: string, params?: Record<string, string>): Promise<TResponse> {
    const url = params
      ? `${this.baseUrl}${path}?${new URLSearchParams(params)}`
      : `${this.baseUrl}${path}`;

    // GETは常に冪等なのでリトライ可能
    return withRetry(async () => {
      const response = await fetch(url);
      if (!response.ok) throw new HttpError(response.status, response.statusText);
      return response.json() as Promise<TResponse>;
    }, this.retryOptions);
  }
}

// リトライバジェット（リトライ嵐防止）
export class RetryBudget {
  private readonly windowMs: number;
  private readonly maxRetryRatio: number;  // リトライ比率の上限
  private requests: number = 0;
  private retries: number = 0;
  private resetAt: number = Date.now();

  constructor(options: { windowMs: number; maxRetryRatio: number }) {
    this.windowMs = options.windowMs;
    this.maxRetryRatio = options.maxRetryRatio;
  }

  recordRequest(): void {
    this.maybeReset();
    this.requests++;
  }

  recordRetry(): boolean {
    this.maybeReset();
    // リトライ比率がバジェットを超えたらリトライ禁止
    const ratio = this.requests > 0 ? this.retries / this.requests : 0;
    if (ratio >= this.maxRetryRatio) {
      logger.warn({ ratio, maxRetryRatio: this.maxRetryRatio }, 'Retry budget exhausted');
      return false;  // リトライ禁止
    }
    this.retries++;
    return true;  // リトライ許可
  }

  private maybeReset(): void {
    if (Date.now() > this.resetAt + this.windowMs) {
      this.requests = 0;
      this.retries = 0;
      this.resetAt = Date.now();
    }
  }
}

// リトライバジェット統合リトライ
const paymentBudget = new RetryBudget({ windowMs: 60_000, maxRetryRatio: 0.1 });  // 10%以上リトライしない

export async function withBudgetedRetry<T>(
  fn: (attempt: number) => Promise<T>,
  budget: RetryBudget,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  budget.recordRequest();

  return withRetry(fn, {
    ...options,
    retryIf: (error) => {
      const defaultShouldRetry = isRetryableError(error);
      if (!defaultShouldRetry) return false;
      return budget.recordRetry();  // バジェットチェック
    },
  });
}

// 使用例: Stripe決済のリトライ
const stripeClient = new IdempotentHttpClient('https://api.stripe.com', {
  maxAttempts: 3,
  baseDelay: 2_000,
  maxDelay: 10_000,
  jitter: 'full',
});

async function chargePayment(orderId: string, amount: number): Promise<string> {
  // orderId をIdempotency-Keyにすることで何度POSTしても1回だけ課金
  const result = await stripeClient.post<ChargeRequest, ChargeResponse>(
    '/v1/charges',
    { amount, currency: 'jpy', description: `Order ${orderId}` },
    { idempotencyKey: `charge-${orderId}`, timeout: 10_000 }
  );
  return result.id;
}
```

---

## まとめ

Claude Codeでリトライパターンを設計する：

1. **CLAUDE.md** に冪等操作のみリトライ・フルジッター付き指数バックオフ・リトライすべきHTTPステータス（429/500/502/503/504）をリスト化・非冪等は絶対リトライしないを明記
2. **フルジッター** でリトライ嵐を防止——全クライアントが`base * 2^attempt`で同時にリトライすると外部APIをさらに叩きつぶす。`random(0, delay)`でバラけさせると負荷が分散
3. **冪等キー（Idempotency-Key）** で副作用の二重実行防止——Stripe等のAPIは同じキーのリクエストを2回受け取っても1回分の処理しかしない。`orderId`を冪等キーにすれば何度リトライしても1回だけ課金
4. **リトライバジェット（最大比率10%）** でリトライ嵐を上流でも防止——全リクエストの10%以上がリトライになったらリトライを止める。単体の指数バックオフだけでは防げない大規模障害に対応

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
