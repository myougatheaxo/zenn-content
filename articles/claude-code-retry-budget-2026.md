---
title: "Claude Codeでリトライ設計を実装する：指数バックオフ・リトライバジェット・Circuit Breaker連携"
emoji: "🔁"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "reliability"]
published: true
published_at: "2026-03-15 18:00"
---

## はじめに

「失敗したらリトライ」——単純なリトライは障害時にかえって負荷を増大させる。指数バックオフ・ジッター・リトライバジェットでシステム全体の過負荷を防ぐ設計をClaude Codeに生成させる。

---

## CLAUDE.mdにリトライ設計ルールを書く

```markdown
## リトライ設計ルール

### 基本方針
- 指数バックオフ: 初回100ms → ×2 → ×4 → ...（最大30秒）
- ジッター: バックオフ時間に±30%のランダム揺らぎ（サンダーリング防止）
- 最大リトライ数: 3回（合計4回試行）

### リトライバジェット
- 1分間のリトライ率: 10%以下を維持
- バジェット超過時は即時失敗（待機なし）
- Redisでリトライ数/総リクエスト数を記録

### リトライ対象の判断
- 5xx系エラー: リトライ対象
- 4xx系エラー（400, 401, 403, 404）: リトライ不要
- ネットワークタイムアウト: リトライ対象
- べき等な操作のみリトライ（POST新規作成は原則リトライ不可）
```

---

## リトライ設計の生成

```
指数バックオフ・リトライバジェット・Circuit Breaker連携を含む
リトライシステムを設計してください。

要件：
- 指数バックオフ+ジッター
- リトライバジェット管理
- Circuit Breaker連携
- リトライ可否の判断ロジック

生成ファイル: src/resilience/retry/
```

---

## 生成されるリトライ実装

```typescript
// src/resilience/retry/backoff.ts — バックオフ計算

export interface BackoffOptions {
  initialDelayMs: number;  // 初回待機時間
  maxDelayMs: number;      // 最大待機時間
  multiplier: number;      // 倍率
  jitterFactor: number;    // ジッター係数（0-1）
}

export const DEFAULT_BACKOFF: BackoffOptions = {
  initialDelayMs: 100,
  maxDelayMs: 30_000,
  multiplier: 2,
  jitterFactor: 0.3,
};

export function calculateBackoff(attempt: number, options: BackoffOptions = DEFAULT_BACKOFF): number {
  // 指数バックオフ
  const exponential = Math.min(
    options.initialDelayMs * Math.pow(options.multiplier, attempt),
    options.maxDelayMs
  );

  // ジッター（±jitterFactor の範囲でランダム揺らぎ）
  const jitterRange = exponential * options.jitterFactor;
  const jitter = (Math.random() * 2 - 1) * jitterRange;

  return Math.max(0, Math.round(exponential + jitter));
}
```

```typescript
// src/resilience/retry/budgetManager.ts — リトライバジェット

export class RetryBudgetManager {
  private readonly windowMs = 60_000; // 1分間

  // リトライを消費してよいかチェック
  async consume(service: string, isRetry: boolean): Promise<boolean> {
    const now = Date.now();
    const bucket = Math.floor(now / this.windowMs);

    const totalKey = `retry:total:${service}:${bucket}`;
    const retryKey = `retry:count:${service}:${bucket}`;

    // パイプラインで原子的に記録
    const pipeline = redis.pipeline();
    pipeline.incr(totalKey);
    pipeline.expire(totalKey, 120);
    if (isRetry) {
      pipeline.incr(retryKey);
      pipeline.expire(retryKey, 120);
    }
    await pipeline.exec();

    if (!isRetry) return true; // 初回試行は常に許可

    // リトライ率チェック
    const [total, retryCount] = await Promise.all([
      redis.get(totalKey).then(v => parseInt(v ?? '1')),
      redis.get(retryKey).then(v => parseInt(v ?? '0')),
    ]);

    const retryRate = retryCount / total;
    if (retryRate > 0.10) { // 10%バジェット超過
      logger.warn({ service, retryRate, retryCount, total }, 'Retry budget exceeded');
      return false;
    }

    return true;
  }

  async getMetrics(service: string): Promise<{ retryRate: number; totalRequests: number }> {
    const bucket = Math.floor(Date.now() / this.windowMs);
    const [total, retries] = await Promise.all([
      redis.get(`retry:total:${service}:${bucket}`).then(v => parseInt(v ?? '0')),
      redis.get(`retry:count:${service}:${bucket}`).then(v => parseInt(v ?? '0')),
    ]);
    return { retryRate: total > 0 ? retries / total : 0, totalRequests: total };
  }
}
```

```typescript
// src/resilience/retry/retryExecutor.ts — リトライ実行エンジン

export interface RetryOptions {
  maxAttempts: number;
  backoff?: Partial<BackoffOptions>;
  service?: string;            // リトライバジェット用サービス名
  shouldRetry?: (error: unknown) => boolean;
}

export class RetryExecutor {
  private readonly budget = new RetryBudgetManager();

  async execute<T>(
    fn: () => Promise<T>,
    options: RetryOptions
  ): Promise<T> {
    const { maxAttempts, service = 'default' } = options;
    const shouldRetry = options.shouldRetry ?? defaultShouldRetry;

    let lastError: unknown;

    for (let attempt = 0; attempt < maxAttempts; attempt++) {
      const isRetry = attempt > 0;

      // リトライバジェットチェック
      if (isRetry) {
        const allowed = await this.budget.consume(service, true);
        if (!allowed) {
          throw new RetryBudgetExceededError(`Retry budget exceeded for ${service}`);
        }
      } else {
        await this.budget.consume(service, false); // 初回: カウントのみ
      }

      try {
        return await fn();
      } catch (error) {
        lastError = error;

        if (!shouldRetry(error)) {
          logger.debug({ attempt, service }, 'Non-retryable error, giving up');
          throw error;
        }

        if (attempt < maxAttempts - 1) {
          const delay = calculateBackoff(attempt, options.backoff);
          logger.debug({ attempt, delay, service }, 'Retrying after backoff');
          await sleep(delay);
        }
      }
    }

    throw lastError;
  }
}

// リトライ可否判断（デフォルト）
function defaultShouldRetry(error: unknown): boolean {
  if (error instanceof HttpError) {
    // 4xx（クライアントエラー）はリトライ不要
    if (error.status >= 400 && error.status < 500) return false;
    // 5xx（サーバーエラー）はリトライ
    if (error.status >= 500) return true;
  }

  if (error instanceof NetworkTimeoutError) return true;
  if (error instanceof ConnectionError) return true;
  if (error instanceof RetryBudgetExceededError) return false;

  return false;
}

// src/resilience/retry/circuitBreaker.ts — Circuit Breaker連携

export class CircuitBreakerWithRetry {
  private readonly executor = new RetryExecutor();
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private failureCount = 0;
  private readonly threshold = 5;   // 5回失敗でopen
  private readonly resetTimeout = 30_000; // 30秒でhalf-open

  async execute<T>(fn: () => Promise<T>, retryOptions: RetryOptions): Promise<T> {
    if (this.state === 'open') {
      throw new CircuitOpenError('Circuit breaker is open');
    }

    try {
      const result = await this.executor.execute(fn, retryOptions);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    this.state = 'closed';
  }

  private onFailure(): void {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'open';
      logger.warn({ failureCount: this.failureCount }, 'Circuit breaker opened');
      setTimeout(() => {
        this.state = 'half-open';
        this.failureCount = Math.floor(this.threshold / 2);
        logger.info('Circuit breaker moved to half-open');
      }, this.resetTimeout);
    }
  }
}

// 使用例: 外部API呼び出し
const retryExecutor = new RetryExecutor();

async function fetchExternalAPI(url: string): Promise<Response> {
  return retryExecutor.execute(
    () => fetch(url, { signal: AbortSignal.timeout(5000) }),
    {
      maxAttempts: 3,
      service: 'external-api',
      backoff: { initialDelayMs: 200, maxDelayMs: 10_000 },
      shouldRetry: (err) => !(err instanceof TypeError), // NetworkError以外はリトライ不可
    }
  );
}
```

---

## まとめ

Claude Codeでリトライ設計を実装する：

1. **CLAUDE.md** に指数バックオフ+ジッター・リトライ率10%上限・5xx系のみリトライ対象を明記
2. **ジッター** でバックオフ時間をランダム揺らぎ——複数クライアントが同時にリトライしてサーバーを再び過負荷にする「サンダーリング・ハード」を防止
3. **リトライバジェット** でRedisのリトライ率を1分ウィンドウで監視——バジェット超過時は即座に失敗させてシステム保護を優先
4. **Circuit Breaker連携** でリトライ+CB統合——繰り返し失敗したサービスへの接続を30秒間遮断しhalf-openで自動復旧

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
