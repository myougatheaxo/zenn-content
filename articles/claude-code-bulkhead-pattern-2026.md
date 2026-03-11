---
title: "Claude Codeでバルクヘッドパターンを実装する：依存サービス障害の隔離・スレッドプール分離"
emoji: "🚢"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "reliability", "architecture"]
published: true
published_at: "2026-03-15 19:00"
---

## はじめに

「外部APIが遅くてサーバー全体がハング」——バルクヘッドパターンは船の隔壁のように依存サービスごとにリソースを隔離し、1つの障害がシステム全体を巻き込まないようにする設計をClaude Codeに生成させる。

---

## CLAUDE.mdにバルクヘッド設計ルールを書く

```markdown
## バルクヘッドパターン設計ルール

### 隔離単位
- 各外部依存サービスに独立した並行リクエスト上限を設定
- コアAPI (同時20件) / 外部API (同時5件) / 重い処理 (同時3件)
- 上限超過: 即座にキューへ（待機上限: 10件）
- キューも満杯: 即座に503を返す

### セマフォ方式（Node.js向け）
- Mutexによる同時実行数制限
- タイムアウト: 10秒（待機タイムアウト）
- メトリクス: 現在の同時実行数・キュー待機数をPrometheusに記録

### ヘルスチェック連動
- 同時実行数が上限の80%超: degraded状態を報告
- Circuit Breakerと組み合わせて二重防御
```

---

## バルクヘッド実装の生成

```
バルクヘッドパターンで外部依存を隔離するシステムを設計してください。

要件：
- セマフォによる同時実行数制限
- サービスごとの独立した制限値
- キューイング（待機上限付き）
- Prometheusメトリクス

生成ファイル: src/resilience/bulkhead/
```

---

## 生成されるバルクヘッド実装

```typescript
// src/resilience/bulkhead/semaphore.ts — セマフォ実装

export class Semaphore {
  private current = 0;
  private readonly queue: Array<() => void> = [];

  constructor(
    readonly name: string,
    readonly maxConcurrent: number,
    readonly maxQueueSize: number,
    readonly waitTimeoutMs: number
  ) {}

  async acquire(): Promise<() => void> {
    if (this.current < this.maxConcurrent) {
      this.current++;
      return this.createRelease();
    }

    // キューが満杯: 即座に拒否
    if (this.queue.length >= this.maxQueueSize) {
      throw new BulkheadFullError(
        `Bulkhead '${this.name}' is full: ${this.current} running, ${this.queue.length} waiting`
      );
    }

    // キューで待機
    return new Promise<() => void>((resolve, reject) => {
      const timer = setTimeout(() => {
        const idx = this.queue.indexOf(ticket);
        if (idx !== -1) this.queue.splice(idx, 1);
        reject(new BulkheadTimeoutError(`Bulkhead '${this.name}' wait timeout after ${this.waitTimeoutMs}ms`));
      }, this.waitTimeoutMs);

      const ticket = () => {
        clearTimeout(timer);
        this.current++;
        resolve(this.createRelease());
      };

      this.queue.push(ticket);
    });
  }

  private createRelease(): () => void {
    let released = false;
    return () => {
      if (released) return;
      released = true;
      this.current--;

      // キューの次のウェイターを起動
      const next = this.queue.shift();
      if (next) next();
    };
  }

  get stats() {
    return { current: this.current, queued: this.queue.length };
  }
}
```

```typescript
// src/resilience/bulkhead/bulkheadRegistry.ts — バルクヘッドレジストリ

interface BulkheadConfig {
  maxConcurrent: number;
  maxQueueSize: number;
  waitTimeoutMs: number;
}

export const BULKHEAD_CONFIGS: Record<string, BulkheadConfig> = {
  // コアDB操作: 余裕を持たせる
  'database': { maxConcurrent: 20, maxQueueSize: 50, waitTimeoutMs: 10_000 },
  // 外部決済API: 厳しく制限（遅い/不安定）
  'stripe': { maxConcurrent: 5, maxQueueSize: 10, waitTimeoutMs: 8_000 },
  // AI/LLM API: 特に厳しく（トークンコスト高い）
  'openai': { maxConcurrent: 3, maxQueueSize: 5, waitTimeoutMs: 30_000 },
  // 外部メール送信
  'email': { maxConcurrent: 10, maxQueueSize: 20, waitTimeoutMs: 5_000 },
  // S3ファイル操作: 並列OK
  's3': { maxConcurrent: 15, maxQueueSize: 30, waitTimeoutMs: 15_000 },
};

export class BulkheadRegistry {
  private readonly bulkheads = new Map<string, Semaphore>();

  constructor() {
    for (const [name, config] of Object.entries(BULKHEAD_CONFIGS)) {
      this.bulkheads.set(name, new Semaphore(name, config.maxConcurrent, config.maxQueueSize, config.waitTimeoutMs));
    }
  }

  get(name: string): Semaphore {
    const bh = this.bulkheads.get(name);
    if (!bh) throw new Error(`Unknown bulkhead: ${name}`);
    return bh;
  }

  // Prometheusメトリクス用
  getAllStats(): Record<string, ReturnType<Semaphore['stats']>> {
    const result: Record<string, any> = {};
    for (const [name, bh] of this.bulkheads) {
      result[name] = bh.stats;
    }
    return result;
  }
}

export const bulkheadRegistry = new BulkheadRegistry();
```

```typescript
// src/resilience/bulkhead/withBulkhead.ts — デコレーター関数

export async function withBulkhead<T>(
  bulkheadName: string,
  fn: () => Promise<T>
): Promise<T> {
  const semaphore = bulkheadRegistry.get(bulkheadName);
  const release = await semaphore.acquire();

  const startTime = Date.now();

  try {
    const result = await fn();
    const duration = Date.now() - startTime;

    // メトリクス記録
    metrics.histogram('bulkhead_execution_ms', duration, { bulkhead: bulkheadName });
    metrics.gauge('bulkhead_concurrent', semaphore.stats.current, { bulkhead: bulkheadName });

    return result;
  } catch (error) {
    metrics.counter('bulkhead_errors_total', 1, {
      bulkhead: bulkheadName,
      error_type: error instanceof BulkheadFullError ? 'full'
        : error instanceof BulkheadTimeoutError ? 'timeout'
        : 'execution',
    });
    throw error;
  } finally {
    release();
  }
}

// 使用例: Stripe決済処理
export class PaymentService {
  async chargeCustomer(customerId: string, amount: number): Promise<ChargeResult> {
    return withBulkhead('stripe', async () => {
      const paymentIntent = await stripe.paymentIntents.create({
        amount,
        currency: 'jpy',
        customer: customerId,
      });
      return { paymentIntentId: paymentIntent.id };
    });
  }
}

// 使用例: AI要約処理
export class SummaryService {
  async summarize(text: string): Promise<string> {
    try {
      return await withBulkhead('openai', async () => {
        const response = await openai.chat.completions.create({
          model: 'gpt-4o-mini',
          messages: [{ role: 'user', content: `以下を100字で要約:\n${text}` }],
        });
        return response.choices[0].message.content ?? '';
      });
    } catch (error) {
      if (error instanceof BulkheadFullError || error instanceof BulkheadTimeoutError) {
        // フォールバック: ルールベース要約
        return text.slice(0, 100) + '...';
      }
      throw error;
    }
  }
}

// ヘルスチェックエンドポイント
router.get('/health/bulkheads', (req, res) => {
  const stats = bulkheadRegistry.getAllStats();
  const degraded = Object.entries(stats).filter(([name, s]) => {
    const config = BULKHEAD_CONFIGS[name];
    return s.current >= config.maxConcurrent * 0.8; // 80%超でdegraded
  });

  res.json({
    status: degraded.length > 0 ? 'degraded' : 'ok',
    bulkheads: stats,
    degraded: degraded.map(([name]) => name),
  });
});
```

---

## まとめ

Claude Codeでバルクヘッドパターンを実装する：

1. **CLAUDE.md** にサービスごとの同時実行数上限・キュー上限・待機タイムアウトを明記——「外部APIはmax5件」のように具体的数値を指定
2. **Semaphore** でNode.jsのイベントループを止めずに同時実行数を制御——acquireがPromiseを返すため、キュー待機もノンブロッキング
3. **キュー満杯は即座に503** を返す——待機を増やすとメモリ圧迫とリクエスト滞留が悪化するため、早めに失敗させてクライアントへ制御を返す
4. **フォールバック** でBulkheadFullError/Timeoutをキャッチしてルールベース代替を返す——AI要約なら一部省略、外部APIなら古いキャッシュを使うなど

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
