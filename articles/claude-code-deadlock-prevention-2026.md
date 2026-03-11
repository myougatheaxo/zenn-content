---
title: "Claude Codeでデッドロック防止を設計する：ロック順序の統一・デッドロック検出・タイムアウトリトライ"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-20 19:00"
---

## はじめに

「本番DBで稀にデッドロックが発生してトランザクションがロールバックされる」「どこでデッドロックが起きているか分からない」——デッドロックの発生メカニズムを理解し、ロック順序の統一・タイムアウト設定・自動リトライで防止する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにデッドロック防止設計ルールを書く

```markdown
## デッドロック防止設計ルール

### デッドロックの原因
- ロック取得順序が一定でない（トランザクションAがテーブル1→2、トランザクションBが2→1）
- 長時間トランザクション（不要なロックを長く保持）
- 過剰なロック範囲（必要以上に多くのレコードをロック）

### 防止策
- ロック順序の統一: 複数テーブル/レコードをロックする場合は常に同じ順序
- SELECT FOR UPDATE SKIP LOCKED: キューワーカーで競合を回避
- lock_timeout設定: デッドロック待ちの最大時間を設定
- 自動リトライ: デッドロック発生時は指数バックオフでリトライ

### PostgreSQL設定
- deadlock_timeout: 1000ms（デフォルト）→ デッドロック検出に1秒かかる
- lock_timeout: 5000ms → 5秒でロック待ちタイムアウト（エラーになりリトライ）
```

---

## デッドロック防止実装の生成

```
デッドロック防止パターンを設計してください。

要件：
- ロック順序の統一
- デッドロック検出とリトライ
- PostgreSQL lock_timeout設定
- デッドロックログ分析

生成ファイル: src/infrastructure/db/
```

---

## 生成されるデッドロック防止実装

```typescript
// src/infrastructure/db/lockOrderedTransaction.ts — ロック順序統一トランザクション

// デッドロック防止の核心: 複数のエンティティを操作する場合は常に同じ順序でロック
// ルール: エンティティIDを昇順ソートしてからFOR UPDATEを実行

export async function withOrderedLocks<T>(
  db: PrismaTransactionClient,
  entities: Array<{ table: string; id: string }>,
  fn: () => Promise<T>
): Promise<T> {
  // IDを昇順でソート（ロック順序を統一してデッドロックを防止）
  const sorted = [...entities].sort((a, b) => {
    // まずテーブル名でソート（テーブル間の順序を固定）
    if (a.table !== b.table) return a.table.localeCompare(b.table);
    // 同じテーブル内はIDでソート
    return a.id.localeCompare(b.id);
  });

  // ソート順でFOR UPDATEを取得
  for (const entity of sorted) {
    await db.$queryRaw`
      SELECT id FROM ${Prisma.raw(entity.table)}
      WHERE id = ${entity.id}
      FOR UPDATE
    `;
  }

  return fn();
}

// 使用例: 在庫更新でデッドロック防止
async function transferInventory(
  fromProductId: string,
  toProductId: string,
  quantity: number
): Promise<void> {
  await prisma.$transaction(async (tx) => {
    // ❌ デッドロックの危険: IDの順序が可変
    // const from = await tx.$queryRaw`SELECT * FROM inventory WHERE id=${fromProductId} FOR UPDATE`;
    // const to   = await tx.$queryRaw`SELECT * FROM inventory WHERE id=${toProductId} FOR UPDATE`;

    // ✅ ロック順序を統一（IDが小さい方を先にロック）
    await withOrderedLocks(tx, [
      { table: 'inventory', id: fromProductId },
      { table: 'inventory', id: toProductId },
    ], async () => {
      await tx.inventory.update({
        where: { id: fromProductId },
        data: { quantity: { decrement: quantity } },
      });
      await tx.inventory.update({
        where: { id: toProductId },
        data: { quantity: { increment: quantity } },
      });
    });
  });
}
```

```typescript
// src/infrastructure/db/deadlockRetry.ts — デッドロック自動リトライ

// PostgreSQLデッドロックエラーコード
const DEADLOCK_ERROR_CODE = '40P01';
const LOCK_TIMEOUT_ERROR_CODE = '55P03';
const SERIALIZATION_FAILURE_CODE = '40001';

export function isDeadlockError(error: unknown): boolean {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    const pgCode = (error.meta as any)?.code;
    return [DEADLOCK_ERROR_CODE, LOCK_TIMEOUT_ERROR_CODE, SERIALIZATION_FAILURE_CODE].includes(pgCode);
  }
  // Prismaがラップしていない場合
  const message = String(error);
  return message.includes('deadlock detected') || message.includes('lock timeout');
}

export async function withDeadlockRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxAttempts?: number;
    baseDelayMs?: number;
    maxDelayMs?: number;
  } = {}
): Promise<T> {
  const { maxAttempts = 3, baseDelayMs = 50, maxDelayMs = 500 } = options;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (!isDeadlockError(error) || attempt === maxAttempts) {
        throw error;
      }

      // ランダムジッター付きバックオフ（全トランザクションが同時にリトライしないよう）
      const delay = Math.min(baseDelayMs * Math.pow(2, attempt - 1), maxDelayMs);
      const jitteredDelay = delay * (0.5 + Math.random() * 0.5);

      logger.warn({
        attempt,
        maxAttempts,
        delayMs: jitteredDelay,
        error: String(error),
      }, 'Deadlock detected, retrying transaction');

      await sleep(jitteredDelay);
    }
  }

  throw new Error('Max deadlock retry attempts exceeded');
}

// PostgreSQL lock_timeout をセッションレベルで設定
export async function withLockTimeout<T>(
  db: PrismaClient,
  timeoutMs: number,
  fn: (tx: PrismaTransactionClient) => Promise<T>
): Promise<T> {
  return withDeadlockRetry(() =>
    db.$transaction(async (tx) => {
      // lock_timeoutでデッドロック待ちを打ち切る（deadlock_timeoutより速い）
      await tx.$executeRaw`SET LOCAL lock_timeout = ${timeoutMs}`;
      // statement_timeoutで長時間トランザクション防止
      await tx.$executeRaw`SET LOCAL statement_timeout = ${timeoutMs * 3}`;
      return fn(tx);
    })
  );
}
```

```typescript
// src/infrastructure/db/deadlockMonitor.ts — デッドロック監視

// PostgreSQLのpg_stat_activityでロック待ちを監視
export class DeadlockMonitor {
  constructor(private readonly db: PrismaClient) {}

  // 現在のロック待ちを取得
  async getLockWaits(): Promise<Array<{
    waitingPid: number;
    waitingQuery: string;
    blockingPid: number;
    waitingDurationMs: number;
  }>> {
    const rows = await this.db.$queryRaw<any[]>`
      SELECT
        waiting.pid AS waiting_pid,
        waiting.query AS waiting_query,
        blocking.pid AS blocking_pid,
        EXTRACT(EPOCH FROM (NOW() - waiting.query_start)) * 1000 AS waiting_duration_ms
      FROM pg_stat_activity AS waiting
      JOIN pg_stat_activity AS blocking
        ON blocking.pid = ANY(pg_blocking_pids(waiting.pid))
      WHERE waiting.wait_event_type = 'Lock'
        AND waiting.pid != pg_backend_pid()
      ORDER BY waiting_duration_ms DESC
    `;

    return rows.map(row => ({
      waitingPid: row.waiting_pid,
      waitingQuery: row.waiting_query,
      blockingPid: row.blocking_pid,
      waitingDurationMs: Math.round(row.waiting_duration_ms),
    }));
  }

  // デッドロック履歴をPGログから集計
  async startMonitoring(intervalMs: number = 30_000): Promise<void> {
    setInterval(async () => {
      const waits = await this.getLockWaits().catch(() => []);

      if (waits.length > 0) {
        logger.warn({ lockWaits: waits }, 'Lock waits detected');

        // 5秒以上待っているロックがあればアラート
        const longWaits = waits.filter(w => w.waitingDurationMs > 5000);
        if (longWaits.length > 0) {
          logger.error({ longWaits }, 'Long-running lock waits - potential deadlock risk');
          metricsClient.gauge('db.lock_waits_count', waits.length);
        }
      }
    }, intervalMs);
  }
}

// 実用的な使用例: 注文確定フロー
async function completeOrder(orderId: string, inventoryItems: Array<{ productId: string; quantity: number }>): Promise<void> {
  // sortedProductIds: IDを昇順ソートしてデッドロック防止
  const sortedItems = [...inventoryItems].sort((a, b) => a.productId.localeCompare(b.productId));

  await withLockTimeout(prisma, 5_000, async (tx) => {
    // ロック取得（ソート済み順）
    for (const item of sortedItems) {
      await tx.$queryRaw`SELECT id FROM inventory WHERE product_id = ${item.productId} FOR UPDATE`;
    }

    // 在庫を一括更新
    for (const item of sortedItems) {
      await tx.inventory.update({
        where: { productId: item.productId },
        data: { available: { decrement: item.quantity } },
      });
    }

    await tx.order.update({ where: { id: orderId }, data: { status: 'completed' } });
  });
}
```

---

## まとめ

Claude Codeでデッドロック防止を設計する：

1. **CLAUDE.md** に複数エンティティのロックは常に同じ順序（IDの昇順）・`lock_timeout`で待ち時間を制限・デッドロック発生時はジッター付きバックオフでリトライを明記
2. **`withOrderedLocks()`でロック順序を統一** ——`[productId-B, productId-A]`を渡しても内部でソートして`A→B`の順にFOR UPDATEを実行。全コードパスで同じ順序でロックを取得することでデッドロックの循環待ちを防止
3. **`lock_timeout`でdeadlock_timeoutより早く検出** ——PostgreSQLのデフォルト`deadlock_timeout=1秒`はロック待ちを1秒続けてからデッドロック検出。`lock_timeout=5秒`に設定するとそれ以前にタイムアウトエラーになりリトライが始まる
4. **デッドロックモニター** でロック待ち状態をリアルタイム監視——`pg_blocking_pids()`で「誰が誰をブロックしているか」を可視化。5秒以上のロック待ちをアラートで検知してDBAが介入できる

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
