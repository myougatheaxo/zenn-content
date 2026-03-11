---
title: "Claude Codeでバックプレッシャーを設計する：過負荷防止・フロー制御・Adaptive Throttling"
emoji: "🚦"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 21:00"
---

## はじめに

「急なトラフィック増加でダウンストリームサービスが崩壊した」——バックプレッシャー（背圧）でコンシューマーが処理できる速度をプロデューサーに伝え、過負荷を防ぐ設計をClaude Codeに生成させる。

---

## CLAUDE.mdにバックプレッシャー設計ルールを書く

```markdown
## バックプレッシャー設計ルール

### 検出
- キューの深さ > 1000: Warning（プロデューサーの速度を下げる）
- キューの深さ > 5000: Critical（リクエストを拒否して503を返す）
- ワーカーのCPU > 80%: スロットリング開始

### 制御
- Adaptive Throttling: エラー率に応じて自動的にレートを下げる
- Token Bucket: バースト許容で平均的なレートを保つ
- Queue Depth Feedback: コンシューマーが深さを報告してプロデューサーを制御

### Shedding（シェディング）
- 過負荷時: 低優先度リクエストを捨てる
- 503 Too Busy を早期に返してクライアントにリトライを促す
- ヘルスチェックAPIは常に応答する（モニタリングを守る）
```

---

## バックプレッシャー実装の生成

```
バックプレッシャーシステムを設計してください。

要件：
- キュー深度モニタリング
- Adaptive Throttling
- Token Bucket
- Load Shedding

生成ファイル: src/resilience/backpressure/
```

---

## 生成されるバックプレッシャー実装

```typescript
// src/resilience/backpressure/adaptiveThrottler.ts — Adaptive Throttling

export class AdaptiveThrottler {
  private acceptRate = 1.0;  // 0.0-1.0（1.0 = 100%受け入れ）
  private readonly minAcceptRate = 0.1;  // 最低10%は受け入れ
  private readonly adjustIntervalMs = 5_000;

  constructor() {
    // 5秒ごとに自動調整
    setInterval(() => this.adjust(), this.adjustIntervalMs);
  }

  private async adjust(): Promise<void> {
    const errorRate = await this.getErrorRate();
    const queueDepth = await this.getQueueDepth();

    let targetRate = this.acceptRate;

    // エラー率に応じてレートを調整
    if (errorRate > 0.1) {
      // 10%超エラー → レートを20%下げる
      targetRate = Math.max(this.minAcceptRate, targetRate * 0.8);
    } else if (errorRate > 0.05) {
      // 5%超エラー → レートを10%下げる
      targetRate = Math.max(this.minAcceptRate, targetRate * 0.9);
    } else if (errorRate < 0.01 && queueDepth < 100) {
      // エラーほぼなし & キュー浅い → レートを少し上げる
      targetRate = Math.min(1.0, targetRate * 1.05);
    }

    if (targetRate !== this.acceptRate) {
      logger.info({ from: this.acceptRate, to: targetRate, errorRate, queueDepth }, 'Throttle rate adjusted');
      this.acceptRate = targetRate;
      metrics.throttleRate.set(targetRate);
    }
  }

  shouldAccept(): boolean {
    return Math.random() < this.acceptRate;
  }

  private async getErrorRate(): Promise<number> {
    const windowSec = 60;
    const now = Math.floor(Date.now() / 1000);
    let totalRequests = 0;
    let totalErrors = 0;

    for (let i = 0; i < windowSec; i++) {
      const bucket = now - i;
      const [req, err] = await Promise.all([
        redis.get(`metrics:requests:${bucket}`).then(v => parseInt(v ?? '0')),
        redis.get(`metrics:errors:${bucket}`).then(v => parseInt(v ?? '0')),
      ]);
      totalRequests += req;
      totalErrors += err;
    }

    return totalRequests > 0 ? totalErrors / totalRequests : 0;
  }

  private async getQueueDepth(): Promise<number> {
    const depth = await redis.xLen('queue:main');
    return depth;
  }
}

// src/resilience/backpressure/tokenBucket.ts — Token Bucket

export class TokenBucket {
  private tokens: number;
  private lastRefillMs: number;

  constructor(
    private readonly capacity: number,     // バケット容量（バーストサイズ）
    private readonly refillRatePerSec: number  // 補充速度（req/s）
  ) {
    this.tokens = capacity;
    this.lastRefillMs = Date.now();
  }

  // トークンを取得（消費）
  tryAcquire(count = 1): boolean {
    this.refill();

    if (this.tokens >= count) {
      this.tokens -= count;
      return true;
    }

    return false; // トークン不足
  }

  private refill(): void {
    const nowMs = Date.now();
    const elapsedSec = (nowMs - this.lastRefillMs) / 1000;
    const tokensToAdd = elapsedSec * this.refillRatePerSec;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefillMs = nowMs;
  }

  get available(): number {
    this.refill();
    return this.tokens;
  }
}

// src/resilience/backpressure/loadShedder.ts — Load Shedding

export interface ShedConfig {
  warningDepth: number;   // キュー深度の警告閾値
  criticalDepth: number;  // キュー深度の危機閾値（ここで503）
}

export class LoadShedder {
  private readonly throttler = new AdaptiveThrottler();
  private readonly bucket = new TokenBucket(100, 50); // バースト100、平常50req/s

  constructor(private readonly config: ShedConfig = { warningDepth: 1000, criticalDepth: 5000 }) {}

  async middleware(req: Request, res: Response, next: NextFunction): Promise<void> {
    // ヘルスチェックは常に通す
    if (req.path === '/health' || req.path === '/metrics') {
      return next();
    }

    const queueDepth = await redis.xLen('queue:main');

    // Critical: 全リクエストを拒否
    if (queueDepth > this.config.criticalDepth) {
      metrics.requestsShed.inc({ reason: 'queue-critical' });
      res.status(503)
        .set('Retry-After', '30')
        .json({ error: 'Service temporarily unavailable', retryAfterSec: 30 });
      return;
    }

    // Warning: Adaptive Throttlingで一部拒否
    if (queueDepth > this.config.warningDepth && !this.throttler.shouldAccept()) {
      metrics.requestsShed.inc({ reason: 'adaptive-throttle' });
      res.status(503)
        .set('Retry-After', '5')
        .json({ error: 'Service overloaded', retryAfterSec: 5 });
      return;
    }

    // Token Bucket: バースト超過
    if (!this.bucket.tryAcquire()) {
      metrics.requestsShed.inc({ reason: 'token-bucket' });
      res.status(429)
        .set('Retry-After', '1')
        .json({ error: 'Too many requests', retryAfterSec: 1 });
      return;
    }

    next();
  }
}

// Expressミドルウェアとして登録
const shedder = new LoadShedder({ warningDepth: 1000, criticalDepth: 5000 });
app.use((req, res, next) => shedder.middleware(req, res, next));

// キュー深度のモニタリングダッシュボード
router.get('/api/admin/backpressure/status', requireAdmin, async (req, res) => {
  const [queueDepth, throttleRate] = await Promise.all([
    redis.xLen('queue:main'),
    redis.get('metrics:throttle-rate').then(v => parseFloat(v ?? '1.0')),
  ]);

  const status =
    queueDepth > 5000 ? 'critical' :
    queueDepth > 1000 ? 'warning' : 'ok';

  res.json({ status, queueDepth, throttleRate, tokenBucketAvailable: shedder['bucket'].available });
});
```

---

## まとめ

Claude Codeでバックプレッシャーを設計する：

1. **CLAUDE.md** にキュー深度1000でWarning/5000でCritical・エラー率に応じてレートを自動調整・過負荷時は503でクライアントにRetry-Afterを返すを明記
2. **Adaptive Throttling** でエラー率が高い時は受け入れ率を自動で下げ、安定したら自動で戻す——人間が介入せずにシステムが過負荷を自律的に管理
3. **Token Bucket** でバーストを許容しつつ平均レートを保つ——capacity=100でリクエストが一気に来ても100まで受け入れ、以降は50req/sの速度で補充
4. **Load Shedding（503 + Retry-After）** でクライアントに「後でリトライせよ」を伝える——ヘルスチェックは例外として常に通し、モニタリングが過負荷に巻き込まれないよう保護

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
