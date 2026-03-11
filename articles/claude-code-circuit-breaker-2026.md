---
title: "Claude Codeでサーキットブレーカーを設計する：障害伝播防止・ハーフオープン・メトリクス連携"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 12:00"
---

## はじめに

「外部APIが落ちたら全リクエストがタイムアウト待ちになって自サービスまで落ちた」——サーキットブレーカーで障害を早期に検出し、フォールバックで自サービスを守る設計をClaude Codeに生成させる。

---

## CLAUDE.mdにサーキットブレーカー設計ルールを書く

```markdown
## サーキットブレーカー設計ルール

### 状態遷移
- CLOSED（正常）: リクエストを通す。エラー率が閾値を超えたらOPENへ
- OPEN（遮断）: リクエストを即座に拒否。タイムアウト後にHALF_OPENへ
- HALF_OPEN（試験）: 1リクエストだけ通す。成功ならCLOSED、失敗ならOPENへ戻る

### 閾値設定
- エラー率閾値: 50%以上でOPEN（最小10リクエスト後から判定）
- OPENタイムアウト: 60秒（60秒後にHALF_OPENで自動回復試行）
- 測定ウィンドウ: 直近60秒のスライディングウィンドウ

### フォールバック
- OPEN時のフォールバック: キャッシュから返す / デフォルト値 / 503を返す
- フォールバックも失敗した場合はエラーをそのままスロー
```

---

## サーキットブレーカー実装の生成

```
サーキットブレーカーを設計してください。

要件：
- CLOSED/OPEN/HALF_OPEN状態遷移
- スライディングウィンドウによるエラー率計算
- Redisによる分散状態共有
- フォールバック対応

生成ファイル: src/resilience/circuitBreaker/
```

---

## 生成されるサーキットブレーカー実装

```typescript
// src/resilience/circuitBreaker/circuitBreaker.ts — サーキットブレーカー

type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

export interface CircuitBreakerOptions {
  name: string;
  failureRateThreshold: number;  // 0.5 = 50%
  minimumRequests: number;       // 最低リクエスト数（少数サンプルで誤検知防止）
  windowSizeMs: number;          // スライディングウィンドウ
  openDurationMs: number;        // OPEN状態の継続時間
}

export class CircuitBreaker {
  private readonly opts: CircuitBreakerOptions;

  // Redis keys
  private get stateKey() { return `cb:state:${this.opts.name}`; }
  private get metricsKey() { return `cb:metrics:${this.opts.name}`; }

  constructor(opts: CircuitBreakerOptions) {
    this.opts = {
      failureRateThreshold: 0.5,
      minimumRequests: 10,
      windowSizeMs: 60_000,
      openDurationMs: 60_000,
      ...opts,
    };
  }

  async execute<T>(fn: () => Promise<T>, fallback?: () => Promise<T>): Promise<T> {
    const state = await this.getState();

    if (state === 'OPEN') {
      logger.debug({ name: this.opts.name }, 'Circuit OPEN — fast failing');
      metrics.circuitBreakerRejected.inc({ name: this.opts.name });

      if (fallback) return fallback();
      throw new CircuitOpenError(`Circuit breaker '${this.opts.name}' is OPEN`);
    }

    if (state === 'HALF_OPEN') {
      // HALF_OPEN: 1リクエストだけ通す（分散環境でもRedis NXで1つに制限）
      const probeAllowed = await redis.set(
        `cb:probe:${this.opts.name}`,
        '1',
        { NX: true, PX: this.opts.openDurationMs }
      );

      if (!probeAllowed) {
        if (fallback) return fallback();
        throw new CircuitOpenError(`Circuit breaker '${this.opts.name}' is HALF_OPEN — probe in progress`);
      }
    }

    const startMs = Date.now();
    try {
      const result = await fn();
      await this.recordSuccess(startMs);
      return result;
    } catch (error) {
      await this.recordFailure(startMs);
      throw error;
    }
  }

  private async getState(): Promise<CircuitState> {
    const stateData = await redis.hGetAll(this.stateKey);

    if (!stateData?.state) return 'CLOSED';

    const state = stateData.state as CircuitState;

    // OPENの場合: タイムアウト経過でHALF_OPENへ
    if (state === 'OPEN') {
      const openedAt = parseInt(stateData.openedAt ?? '0');
      if (Date.now() - openedAt > this.opts.openDurationMs) {
        await redis.hSet(this.stateKey, 'state', 'HALF_OPEN');
        logger.info({ name: this.opts.name }, 'Circuit transitioned to HALF_OPEN');
        return 'HALF_OPEN';
      }
    }

    return state;
  }

  private async recordSuccess(startMs: number): Promise<void> {
    const durationMs = Date.now() - startMs;
    const bucket = Math.floor(Date.now() / 1000); // 1秒バケット

    await redis.hIncrBy(this.metricsKey, `success:${bucket}`, 1);
    await redis.expire(this.metricsKey, Math.ceil(this.opts.windowSizeMs / 1000) * 2);

    metrics.circuitBreakerLatency.observe({ name: this.opts.name, state: 'success' }, durationMs / 1000);

    const currentState = await redis.hGet(this.stateKey, 'state');
    if (currentState === 'HALF_OPEN') {
      // 試験成功: CLOSEDへ
      await redis.hSet(this.stateKey, { state: 'CLOSED', closedAt: Date.now().toString() });
      await redis.del(`cb:probe:${this.opts.name}`);
      logger.info({ name: this.opts.name }, 'Circuit CLOSED — service recovered');
    }
  }

  private async recordFailure(startMs: number): Promise<void> {
    const durationMs = Date.now() - startMs;
    const bucket = Math.floor(Date.now() / 1000);

    await redis.hIncrBy(this.metricsKey, `failure:${bucket}`, 1);
    await redis.expire(this.metricsKey, Math.ceil(this.opts.windowSizeMs / 1000) * 2);

    metrics.circuitBreakerLatency.observe({ name: this.opts.name, state: 'failure' }, durationMs / 1000);

    const currentState = await redis.hGet(this.stateKey, 'state');
    if (currentState === 'HALF_OPEN') {
      // 試験失敗: OPENへ戻る
      await redis.hSet(this.stateKey, { state: 'OPEN', openedAt: Date.now().toString() });
      await redis.del(`cb:probe:${this.opts.name}`);
      logger.warn({ name: this.opts.name }, 'Circuit re-OPENED — probe failed');
      return;
    }

    // エラー率チェック
    await this.checkFailureRate();
  }

  private async checkFailureRate(): Promise<void> {
    const allMetrics = await redis.hGetAll(this.metricsKey);
    const nowSec = Math.floor(Date.now() / 1000);
    const windowSec = Math.ceil(this.opts.windowSizeMs / 1000);

    let successes = 0;
    let failures = 0;

    for (const [key, value] of Object.entries(allMetrics)) {
      const [type, bucketStr] = key.split(':');
      const bucket = parseInt(bucketStr);

      if (nowSec - bucket <= windowSec) {
        if (type === 'success') successes += parseInt(value);
        if (type === 'failure') failures += parseInt(value);
      }
    }

    const total = successes + failures;
    if (total < this.opts.minimumRequests) return; // サンプル不足

    const failureRate = failures / total;
    if (failureRate >= this.opts.failureRateThreshold) {
      await redis.hSet(this.stateKey, { state: 'OPEN', openedAt: Date.now().toString() });
      logger.warn(
        { name: this.opts.name, failureRate, total },
        `Circuit OPENED — failure rate ${(failureRate * 100).toFixed(1)}%`
      );
    }
  }

  async getStatus(): Promise<{
    state: CircuitState;
    failureRate: number;
    totalRequests: number;
  }> {
    const state = await this.getState();
    const allMetrics = await redis.hGetAll(this.metricsKey);
    const nowSec = Math.floor(Date.now() / 1000);
    const windowSec = Math.ceil(this.opts.windowSizeMs / 1000);

    let successes = 0;
    let failures = 0;
    for (const [key, value] of Object.entries(allMetrics)) {
      const [type, bucketStr] = key.split(':');
      if (nowSec - parseInt(bucketStr) <= windowSec) {
        if (type === 'success') successes += parseInt(value);
        if (type === 'failure') failures += parseInt(value);
      }
    }

    const total = successes + failures;
    return { state, failureRate: total > 0 ? failures / total : 0, totalRequests: total };
  }
}

// 使用例
const paymentCircuit = new CircuitBreaker({ name: 'payment-service', failureRateThreshold: 0.5, minimumRequests: 10, windowSizeMs: 60_000, openDurationMs: 60_000 });

const result = await paymentCircuit.execute(
  () => paymentApi.charge(amount),
  () => {
    // フォールバック: キャッシュから返す or 503
    throw new ServiceUnavailableError('Payment service unavailable');
  }
);
```

---

## まとめ

Claude Codeでサーキットブレーカーを設計する：

1. **CLAUDE.md** にCLOSED/OPEN/HALF_OPENの3状態・エラー率50%でOPEN・60秒後にHALF_OPENで自動回復試行・スライディングウィンドウ60秒を明記
2. **Redis分散状態** でマルチインスタンス環境でも共有——1台のWorkerがOPENに遷移すると全Workerが即座にOPENを認識してfast-failする
3. **HALF_OPENの排他制御** `SET NX`で「1インスタンスだけがプローブリクエストを送る」——複数インスタンスが同時に回復試行してバックエンドを圧迫しない
4. **最小リクエスト数（minimumRequests）** で少数サンプルでの誤検知防止——10リクエスト未満では`failureRate=1.0`でも遮断しない

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
