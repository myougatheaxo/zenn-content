---
title: "Claude Codeで並行処理制限を設計する：セマフォ・Mutex・Promise同時実行数制御・バースト制御"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-21 18:00"
---

## はじめに

「Promise.allで1000件を一度に送信してAPIがタイムアウトした」「同じリソースに対して並行書き込みが発生してデータが壊れた」——セマフォとMutexで並行処理数を制御し、リソースを保護する設計をClaude Codeに生成させる。

---

## CLAUDE.mdに並行処理制限設計ルールを書く

```markdown
## 並行処理制限設計ルール

### 並行処理の問題
- 無制限のPromise.all: 外部APIレート制限、DB接続枯渇
- 競合状態: 同一リソースへの同時書き込みでデータ不整合
- メモリ不足: 大量の並行タスクがメモリを消費

### セマフォ vs Mutex
- Semaphore: N個まで並行許可（DB接続5本まで、API呼び出し10並行まで）
- Mutex: 1個だけ許可（書き込み操作の排他制御）
- 実装: Promiseチェーンで実現（OS/CPUのプリミティブは不要）

### 実用パターン
- pLimit: 並行数制限ライブラリ（軽量で使いやすい）
- Chunk + delayで外部API呼び出しをレート制限
- Redis Lua atomicで分散セマフォ
```

---

## 並行処理制限実装の生成

```
並行処理制限パターンを設計してください。

要件：
- インプロセスセマフォ（Promise）
- 分散セマフォ（Redis）
- 並行数制限付きバッチ処理
- デッドロック防止

生成ファイル: src/shared/concurrency/
```

---

## 生成される並行処理制限実装

```typescript
// src/shared/concurrency/semaphore.ts — セマフォ実装

export class Semaphore {
  private readonly queue: Array<(release: () => void) => void> = [];
  private running = 0;

  constructor(private readonly maxConcurrency: number) {}

  // セマフォを取得（最大並行数に達したら待機）
  async acquire(): Promise<() => void> {
    return new Promise((resolve) => {
      if (this.running < this.maxConcurrency) {
        this.running++;
        resolve(this.createRelease());
      } else {
        // キューに追加して待機
        this.queue.push((release) => resolve(release));
      }
    });
  }

  // 自動リリース付き実行
  async run<T>(fn: () => Promise<T>): Promise<T> {
    const release = await this.acquire();
    try {
      return await fn();
    } finally {
      release();
    }
  }

  private createRelease(): () => void {
    let released = false;
    return () => {
      if (released) return;
      released = true;

      if (this.queue.length > 0) {
        const next = this.queue.shift()!;
        next(this.createRelease());
      } else {
        this.running--;
      }
    };
  }

  get availableSlots(): number {
    return Math.max(0, this.maxConcurrency - this.running);
  }

  get waitingCount(): number {
    return this.queue.length;
  }
}

// Mutex（Semaphore(1)の特殊ケース）
export class Mutex {
  private readonly semaphore = new Semaphore(1);

  async lock(): Promise<() => void> {
    return this.semaphore.acquire();
  }

  async withLock<T>(fn: () => Promise<T>): Promise<T> {
    return this.semaphore.run(fn);
  }
}
```

```typescript
// src/shared/concurrency/limitedConcurrency.ts — 並行数制限バッチ処理

// 並行数を制限したPromise.all
export async function pLimit<T>(
  tasks: Array<() => Promise<T>>,
  concurrency: number
): Promise<T[]> {
  const semaphore = new Semaphore(concurrency);
  return Promise.all(tasks.map(task => semaphore.run(task)));
}

// 順序保証が必要な場合（入力順に結果を返す）
export async function pLimitOrdered<T, R>(
  items: T[],
  fn: (item: T, index: number) => Promise<R>,
  concurrency: number
): Promise<R[]> {
  const semaphore = new Semaphore(concurrency);
  return Promise.all(
    items.map((item, index) =>
      semaphore.run(() => fn(item, index))
    )
  );
}

// バースト制御付き（レート制限のあるAPIへの一括送信）
export async function rateLimitedBatch<T, R>(
  items: T[],
  fn: (item: T) => Promise<R>,
  options: {
    concurrency: number;
    delayBetweenBatchesMs?: number;
    batchSize?: number;
  }
): Promise<R[]> {
  const { concurrency, delayBetweenBatchesMs = 0, batchSize = concurrency } = options;
  const results: R[] = [];

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await pLimit(batch.map(item => () => fn(item)), concurrency);
    results.push(...batchResults);

    if (delayBetweenBatchesMs > 0 && i + batchSize < items.length) {
      await sleep(delayBetweenBatchesMs);
    }
  }

  return results;
}

// 実用例: メール一括送信（同時10件、バッチ間500ms待機）
async function sendBulkEmails(users: User[]): Promise<void> {
  const results = await rateLimitedBatch(
    users,
    async (user) => emailService.send(user.email, 'newsletter'),
    { concurrency: 10, delayBetweenBatchesMs: 500, batchSize: 50 }
  );

  const failed = results.filter(r => r.status === 'failed');
  logger.info({ total: users.length, failed: failed.length }, 'Bulk email completed');
}
```

```typescript
// src/shared/concurrency/distributedSemaphore.ts — Redis分散セマフォ

export class DistributedSemaphore {
  private readonly key: string;
  private readonly acquireScript: string;
  private readonly releaseScript: string;

  constructor(
    private readonly redis: Redis,
    private readonly name: string,
    private readonly maxConcurrency: number,
    private readonly ttlMs: number = 30_000  // ロックの最大保持時間
  ) {
    this.key = `semaphore:${name}`;

    // Lua atomicスクリプト: 現在の使用数をチェックして取得
    this.acquireScript = `
      local current = tonumber(redis.call('get', KEYS[1]) or 0)
      if current < tonumber(ARGV[1]) then
        redis.call('incr', KEYS[1])
        redis.call('pexpire', KEYS[1], ARGV[2])
        return 1
      end
      return 0
    `;

    // 解放スクリプト
    this.releaseScript = `
      local current = tonumber(redis.call('get', KEYS[1]) or 0)
      if current > 0 then
        redis.call('decr', KEYS[1])
      end
      return 1
    `;
  }

  async acquire(timeoutMs: number = 10_000): Promise<() => Promise<void>> {
    const deadline = Date.now() + timeoutMs;
    const instanceId = ulid();

    while (Date.now() < deadline) {
      const acquired = await this.redis.eval(
        this.acquireScript,
        1,
        this.key,
        String(this.maxConcurrency),
        String(this.ttlMs)
      ) as number;

      if (acquired === 1) {
        // 取得成功 → リリース関数を返す
        return async () => {
          await this.redis.eval(this.releaseScript, 1, this.key);
        };
      }

      // 待機してリトライ
      await sleep(50 + Math.random() * 50);
    }

    throw new SemaphoreTimeoutError(this.name, timeoutMs);
  }

  async run<T>(fn: () => Promise<T>, timeoutMs?: number): Promise<T> {
    const release = await this.acquire(timeoutMs);
    try {
      return await fn();
    } finally {
      await release();
    }
  }

  async getCurrentCount(): Promise<number> {
    const count = await this.redis.get(this.key);
    return parseInt(count ?? '0');
  }
}

// 使用例: 外部決済APIへの同時アクセスをマルチインスタンスで5件制限
const paymentSemaphore = new DistributedSemaphore(redis, 'payment-gateway', 5, 30_000);

async function processPayment(orderId: string): Promise<void> {
  await paymentSemaphore.run(async () => {
    // ここは全インスタンス合わせて最大5並行
    await stripeClient.createCharge(orderId);
  }, 15_000);
}

// タイムアウト付きMutex（ファイル書き込みの排他制御）
const fileWriteMutex = new Mutex();

async function appendToLog(message: string): Promise<void> {
  await fileWriteMutex.withLock(async () => {
    await fs.appendFile('/var/log/app.log', message + '\n');
  });
}
```

---

## まとめ

Claude Codeで並行処理制限を設計する：

1. **CLAUDE.md** にPromise.allは並行数制限必須・Semaphore(N)でN並行まで許可・Mutex=Semaphore(1)で排他制御・外部APIへのバッチはバッチ間delayでバースト防止を明記
2. **`Semaphore.run(fn)`** で並行数を自動管理——`await semaphore.run(() => expensiveOperation())`だけで最大並行数を制限。`acquire()`/`release()`のペアを忘れるリスクがなく、エラー時もfinallyでリリースされる
3. **`rateLimitedBatch()`** で外部APIへの一括送信を安全に——1000件のメールを`concurrency:10, delayBetweenBatchesMs:500`で送ると、50件ずつ同時10件で送りながら500ms休憩。SESなどの1秒あたりのレート制限に対応
4. **分散セマフォ（Redis Lua）** でマルチインスタンス間の並行数を制限——インプロセスのSemaphoreは単一インスタンスにしか効かない。Redis `incr`/`decr`をLuaで原子実行することで複数Podにまたがって並行数を制限

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
