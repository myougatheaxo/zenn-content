---
title: "Claude CodeでPostgreSQLペシミスティックロックを設計する：SELECT FOR UPDATE・デッドロック防止"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "prisma"]
published: true
published_at: "2026-03-16 12:00"
---

## はじめに

「在庫の競合更新でオーバーセールが発生した」「残高がマイナスになった」——楽観的ロックで対処できない強い競合にはSELECT FOR UPDATEによるペシミスティックロックが必要。デッドロックを防ぎながら実装する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにペシミスティックロック設計ルールを書く

```markdown
## ペシミスティックロック設計ルール

### 使用基準
- 在庫・残高など競合が高頻度で発生する更新: SELECT FOR UPDATE
- 楽観的ロック（version番号）ではリトライが多発する: ペシミスティックに切り替え
- 読み取りだけなら不要（SELECT FOR SHAREを検討）

### デッドロック防止
- 複数行をロックする場合は必ず同じ順序でロック取得（例: id昇順）
- ロック保持時間を最小化（長いビジネスロジックをロック内に入れない）
- NOWAIT/SKIP LOCKEDで即座に失敗させてリトライ制御

### タイムアウト設定
- SET LOCAL lock_timeout = '5s'（トランザクション単位で設定）
- デッドロックは自動検出（deadlock_timeout = 1s）してエラー → リトライ
```

---

## ペシミスティックロック実装の生成

```
PostgreSQLのSELECT FOR UPDATEを使ったペシミスティックロックを設計してください。

要件：
- Prismaでのraw SQL LOCK取得
- デッドロック防止（IDソート順ロック）
- NOWAIT/SKIP LOCKED
- ロックタイムアウト
- リトライラッパー

生成ファイル: src/db/locking/
```

---

## 生成されるペシミスティックロック実装

```typescript
// src/db/locking/pessimisticLock.ts — ロック取得ヘルパー

type LockMode = 'UPDATE' | 'SHARE' | 'UPDATE NOWAIT' | 'UPDATE SKIP LOCKED';

export async function lockRows<T extends { id: string }>(
  tx: Prisma.TransactionClient,
  table: string,
  ids: string[],
  mode: LockMode = 'UPDATE'
): Promise<void> {
  if (ids.length === 0) return;

  // デッドロック防止: 常にID昇順でロック取得
  const sortedIds = [...ids].sort();

  await tx.$executeRawUnsafe(
    `SELECT id FROM "${table}" WHERE id = ANY($1::uuid[]) ORDER BY id FOR ${mode}`,
    sortedIds
  );
}

// トランザクション単位でロックタイムアウトを設定
export async function withLockTimeout<T>(
  client: PrismaClient,
  timeoutMs: number,
  fn: (tx: Prisma.TransactionClient) => Promise<T>
): Promise<T> {
  return client.$transaction(async (tx) => {
    await tx.$executeRawUnsafe(`SET LOCAL lock_timeout = '${timeoutMs}ms'`);
    return fn(tx);
  });
}
```

```typescript
// src/db/locking/inventoryService.ts — 在庫更新の実例

export class InventoryService {
  // 在庫の競合更新（オーバーセール防止）
  async deductInventory(
    orderItems: Array<{ productId: string; quantity: number }>
  ): Promise<void> {
    // ロックタイムアウト5秒以内でトランザクション実行
    await withLockTimeout(prisma, 5000, async (tx) => {
      const productIds = orderItems.map(i => i.productId);

      // SELECT FOR UPDATE（IDソート順でデッドロック防止）
      await lockRows(tx, 'Product', productIds, 'UPDATE');

      // ロック取得後に現在の在庫を取得
      const products = await tx.product.findMany({
        where: { id: { in: productIds } },
        select: { id: true, inventory: true, name: true },
      });

      // 在庫チェック（ロック後なので安全）
      for (const item of orderItems) {
        const product = products.find(p => p.id === item.productId);
        if (!product) throw new NotFoundError(`Product ${item.productId} not found`);
        if (product.inventory < item.quantity) {
          throw new InsufficientInventoryError(
            `Insufficient inventory for ${product.name}: need ${item.quantity}, have ${product.inventory}`
          );
        }
      }

      // 在庫更新（競合なし、ロック内なので安全）
      await Promise.all(orderItems.map(item =>
        tx.product.update({
          where: { id: item.productId },
          data: { inventory: { decrement: item.quantity } },
        })
      ));
    });
  }

  // SKIP LOCKED: キューワーカーパターン（処理中の行をスキップ）
  async dequeueForProcessing(): Promise<Order | null> {
    return prisma.$transaction(async (tx) => {
      // 他のワーカーが処理中の行をスキップして次の行を取得
      const rows = await tx.$queryRaw<Array<{ id: string }>>`
        SELECT id FROM "Order"
        WHERE status = 'pending'
          AND deleted_at IS NULL
        ORDER BY created_at ASC
        LIMIT 1
        FOR UPDATE SKIP LOCKED  -- 他のトランザクションがロックしている行はスキップ
      `;

      if (rows.length === 0) return null;

      // 処理開始マーク（ロック内なので安全）
      return tx.order.update({
        where: { id: rows[0].id },
        data: { status: 'processing', processingStartedAt: new Date() },
      });
    });
  }
}
```

```typescript
// src/db/locking/retryOnLockError.ts — デッドロック・ロックタイムアウト時のリトライ

const RETRYABLE_PG_CODES = new Set([
  '40001', // serialization_failure
  '40P01', // deadlock_detected
  '55P03', // lock_not_available (NOWAIT)
]);

export async function withLockRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  baseDelayMs = 50
): Promise<T> {
  let lastError: unknown;

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      const pgCode = error?.code ?? error?.cause?.code;

      if (RETRYABLE_PG_CODES.has(pgCode)) {
        lastError = error;
        const delay = baseDelayMs * Math.pow(2, attempt) * (0.5 + Math.random() * 0.5); // ジッター付き
        logger.warn({ attempt, pgCode, delayMs: delay }, 'Lock contention, retrying...');
        await sleep(delay);
      } else {
        throw error; // リトライ不可能なエラーは即re-throw
      }
    }
  }

  throw lastError;
}

// 使用例: デッドロック自動リトライ付き在庫更新
async function placeOrder(userId: string, items: OrderItem[]): Promise<Order> {
  return withLockRetry(async () => {
    await inventoryService.deductInventory(items);
    return orderService.create(userId, items);
  });
}

// 楽観的ロックとの比較（参考）
export async function optimisticUpdate(productId: string, quantity: number): Promise<void> {
  const MAX_RETRIES = 10;
  for (let i = 0; i < MAX_RETRIES; i++) {
    const product = await prisma.product.findUniqueOrThrow({ where: { id: productId } });

    try {
      await prisma.product.update({
        where: { id: productId, version: product.version }, // versionチェック
        data: { inventory: { decrement: quantity }, version: { increment: 1 } },
      });
      return;
    } catch {
      if (i === MAX_RETRIES - 1) throw new Error('Too many conflicts (optimistic lock)');
      await sleep(10);
    }
  }
}
// → 競合が高頻度なら楽観的ロックのリトライが多発 → ペシミスティックに切り替え
```

---

## まとめ

Claude CodeでPostgreSQLペシミスティックロックを設計する：

1. **CLAUDE.md** に使用基準（高頻度競合）・ID昇順ロック必須・lock_timeout 5s設定・NOWAIT/SKIP LOCKED使い分けを明記
2. **ID昇順ソートでロック取得** ——複数行をロックする際に順序を統一することでデッドロックの発生を根本的に防止
3. **SKIP LOCKED** で複数ワーカーが同じキューを処理可能——他のワーカーが処理中の行をスキップして次の行へ進む
4. **デッドロック/シリアライゼーションエラー（40P01/40001）** を検出して自動リトライ——ジッター付き指数バックオフで競合を分散

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
