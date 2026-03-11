---
title: "Claude Codeで分散ロックを設計する：Redlock・デッドロック防止・冪等処理"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "distributed"]
published: true
published_at: "2026-03-12 11:00"
---

## はじめに

複数サーバーで同じジョブを同時実行してしまう——Redis Redlockアルゴリズムで分散ロックを実装し、クリティカルセクションを排他制御する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdに分散ロック設計ルールを書く

```markdown
## 分散ロック設計ルール

### Redlockアルゴリズム
- Redisは必ず奇数台（3台以上）使用
- ロック有効期限: 処理時間の2倍を設定
- 取得タイムアウト: 100ms以内（取得できなければ諦める）

### デッドロック防止
- TTLで自動解放（プロセスがクラッシュしても詰まらない）
- 同一スレッドの再入可能ロックは再帰カウンターで管理
- ロックIDを検証してから解放（他プロセスのロックを解放しない）

### エラーハンドリング
- ロック取得失敗は再試行しない（競合が多い場合はアーキテクチャを見直す）
- タイムアウトで自動解放されるため、クリーンアップは不要
```

---

## 分散ロック実装の生成

```
Redisを使った分散ロックを設計してください。

要件：
- Redlockアルゴリズム（Redis 3台）
- 自動TTLによるデッドロック防止
- タイムアウト付きロック取得
- ロック保持中の処理失敗対応

生成ファイル: src/lock/
```

---

## 生成される分散ロック実装

```typescript
// src/lock/distributedLock.ts
import Redlock from 'redlock';
import Redis from 'ioredis';

// Redisクライアント（本番は3台以上の独立したRedisインスタンス）
const redisClients = [
  new Redis(process.env.REDIS_URL_1!),
  new Redis(process.env.REDIS_URL_2!),
  new Redis(process.env.REDIS_URL_3!),
];

const redlock = new Redlock(redisClients, {
  driftFactor: 0.01,    // TTLの1%を時刻ずれとして考慮
  retryCount: 3,         // 3回試みてダメなら失敗
  retryDelay: 200,       // 200ms待って再試行
  retryJitter: 100,      // 0-100msのランダムジッター（サンダーリング・ハード防止）
  automaticExtensionThreshold: 500, // TTL残り500ms以下で自動延長
});

// ===========================
// 型安全なロックヘルパー
// ===========================

export class DistributedLock {
  // 排他ロックで処理を実行（取得できなければThrow）
  static async withLock<T>(
    resource: string,
    ttlMs: number,
    fn: () => Promise<T>
  ): Promise<T> {
    const lockKey = `lock:${resource}`;
    let lock: Redlock.Lock | null = null;

    try {
      lock = await redlock.acquire([lockKey], ttlMs);
      return await fn();
    } finally {
      if (lock) {
        await lock.release().catch(err => {
          // ロック解放失敗はTTLで自動解放されるので無視
          logger.warn({ err, resource }, 'Failed to release lock (will auto-expire)');
        });
      }
    }
  }

  // ロックを試みる（取得できない場合はスキップ）
  static async tryWithLock<T>(
    resource: string,
    ttlMs: number,
    fn: () => Promise<T>
  ): Promise<{ executed: boolean; result?: T }> {
    const lockKey = `lock:${resource}`;

    try {
      const lock = await redlock.acquire([lockKey], ttlMs, { retryCount: 0 });
      try {
        const result = await fn();
        return { executed: true, result };
      } finally {
        await lock.release().catch(() => {});
      }
    } catch (err) {
      if (err instanceof Redlock.ResourceLockedError) {
        return { executed: false }; // 他プロセスが実行中→スキップ
      }
      throw err;
    }
  }
}

// ===========================
// 具体的な使用例
// ===========================

// 1. クーポン使用（在庫なし二重使用防止）
export async function redeemCoupon(couponCode: string, userId: string): Promise<void> {
  await DistributedLock.withLock(
    `coupon:${couponCode}`,
    5_000, // 5秒TTL（処理時間の2倍）
    async () => {
      const coupon = await prisma.coupon.findUniqueOrThrow({ where: { code: couponCode } });

      if (coupon.usedAt) throw new ConflictError('Coupon already used');
      if (coupon.expiresAt < new Date()) throw new ValidationError('Coupon expired');

      await prisma.coupon.update({
        where: { id: coupon.id },
        data: { usedAt: new Date(), usedBy: userId },
      });
    }
  );
}

// 2. バッチジョブ（複数インスタンスで同じジョブを実行しない）
export async function processDailyReport(): Promise<void> {
  const today = new Date().toISOString().split('T')[0];

  const { executed } = await DistributedLock.tryWithLock(
    `daily-report:${today}`,
    10 * 60 * 1000, // 10分TTL
    async () => {
      logger.info('Processing daily report...');
      await generateAndSendDailyReport();
    }
  );

  if (!executed) {
    logger.info('Daily report already being processed by another instance, skipping');
  }
}

// 3. フラッシュセール（在庫の同時更新防止）
export async function purchaseFlashSale(productId: string, userId: string): Promise<Order> {
  return DistributedLock.withLock(
    `flash-sale:${productId}`,
    3_000,
    async () => {
      const product = await prisma.product.findUniqueOrThrow({ where: { id: productId } });

      if (product.flashSaleStock <= 0) {
        throw new ConflictError('Flash sale sold out');
      }

      // デクリメントは楽観的ロック（SELECT FOR UPDATEはロック内不要）
      const updated = await prisma.product.updateMany({
        where: { id: productId, flashSaleStock: { gt: 0 } },
        data: { flashSaleStock: { decrement: 1 } },
      });

      if (updated.count === 0) throw new ConflictError('Flash sale sold out');

      return prisma.order.create({
        data: { userId, productId, type: 'flash_sale' },
      });
    }
  );
}
```

---

## ロック延長（長時間処理）

```typescript
// src/lock/extendableLock.ts
// 処理が長引く場合にロックを延長する

export class ExtendableLock {
  private lock: Redlock.Lock;
  private extendTimer: NodeJS.Timeout | null = null;

  constructor(
    private readonly resource: string,
    private readonly ttlMs: number
  ) {}

  async acquire(): Promise<void> {
    this.lock = await redlock.acquire([`lock:${this.resource}`], this.ttlMs);

    // TTLの半分になったら自動延長
    this.extendTimer = setInterval(async () => {
      try {
        this.lock = await this.lock.extend(this.ttlMs);
        logger.debug({ resource: this.resource }, 'Lock extended');
      } catch (err) {
        logger.error({ err, resource: this.resource }, 'Failed to extend lock');
        this.stopExtending();
      }
    }, this.ttlMs / 2);
  }

  async release(): Promise<void> {
    this.stopExtending();
    await this.lock.release().catch(() => {});
  }

  private stopExtending(): void {
    if (this.extendTimer) {
      clearInterval(this.extendTimer);
      this.extendTimer = null;
    }
  }
}

// 使用例: 大きなCSVエクスポート処理（時間がかかる）
export async function exportLargeDataset(exportId: string): Promise<void> {
  const lock = new ExtendableLock(`export:${exportId}`, 30_000);
  await lock.acquire();

  try {
    await processLargeExport(exportId); // 数分かかる可能性がある処理
  } finally {
    await lock.release();
  }
}
```

---

## まとめ

Claude Codeで分散ロックを設計する：

1. **CLAUDE.md** にRedis奇数台・TTL処理時間2倍・取得失敗は再試行しないを明記
2. **withLock** でfinally内で必ず解放（TTLもフォールバックとして機能）
3. **tryWithLock** でスキップパターン（バッチジョブの重複実行防止）
4. **ExtendableLock** でTTL自動延長（長時間処理が途中でロック期限切れしない）

---

*分散ロック設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
