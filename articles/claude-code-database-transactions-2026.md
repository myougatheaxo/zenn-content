---
title: "Claude CodeでDBトランザクションを設計する：楽観的ロック・デッドロック・Saga"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "prisma"]
published: true
---

## はじめに

トランザクションの使い方を間違えるとデッドロックが発生する、在庫がマイナスになる、注文が二重作成される。Claude Codeに安全なトランザクション設計を生成させる。

---

## CLAUDE.mdにトランザクションルールを書く

```markdown
## DBトランザクション設計ルール

### 基本原則
- 複数テーブルへの書き込みは必ずトランザクション
- トランザクション内でのI/O（HTTP呼び出し等）禁止
- トランザクションは短く保つ（長いとロック競合の原因）

### ロック戦略
- 競合頻度低: 楽観的ロック（version番号チェック）
- 競合頻度高: SELECT FOR UPDATE（悲観的ロック）
- 在庫・残高: 必ず楽観的または悲観的ロックで整合性保証

### デッドロック防止
- 複数テーブルのロック順序を常に同一にする
- SELECT FOR UPDATE は最後のテーブルにのみ使う
- タイムアウトを設定（5秒）

### 分散トランザクション
- 外部サービス（Stripe等）を含む場合はSagaパターン
- 補償トランザクション（ロールバック相当の操作）を用意
```

---

## 楽観的ロックの実装

```
在庫の楽観的ロック実装を生成してください。

シナリオ: 注文作成時に在庫を減らす
問題: 同時リクエストで在庫がマイナスになる

要件：
- versionカラムで楽観的ロック
- 競合時は3回リトライ（指数バックオフ）
- 在庫0以下は拒否
```

---

## 生成される楽観的ロック実装

```typescript
// src/services/inventoryService.ts

// Prismaスキーマ
// model Inventory {
//   id       String @id
//   quantity Int
//   version  Int    @default(0)
// }

async function decrementInventoryOptimistic(
  productId: string,
  quantity: number,
  maxRetries = 3
): Promise<void> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    // 現在の在庫を取得
    const inventory = await prisma.inventory.findUnique({
      where: { id: productId },
    });

    if (!inventory || inventory.quantity < quantity) {
      throw new ConflictError('Insufficient inventory');
    }

    // バージョン番号で楽観的ロック
    const result = await prisma.inventory.updateMany({
      where: {
        id: productId,
        version: inventory.version, // 取得時のバージョンと一致する場合のみ更新
        quantity: { gte: quantity }, // 在庫が十分ある場合のみ
      },
      data: {
        quantity: { decrement: quantity },
        version: { increment: 1 },
      },
    });

    if (result.count === 1) {
      return; // 更新成功
    }

    // 競合が発生（他のリクエストが先に更新した）
    if (attempt < maxRetries - 1) {
      await new Promise(resolve =>
        setTimeout(resolve, Math.pow(2, attempt) * 100) // 100ms, 200ms, 400ms
      );
    }
  }

  throw new ConflictError('Inventory update failed after retries — try again');
}
```

---

## 悲観的ロック（SELECT FOR UPDATE）

```typescript
// 在庫の悲観的ロック（競合頻度が高い場合）
async function decrementInventoryPessimistic(
  productId: string,
  quantity: number
): Promise<void> {
  await prisma.$transaction(async (tx) => {
    // 行ロックを取得（他のトランザクションを待機させる）
    const inventory = await tx.$queryRaw<Array<{quantity: number}>>`
      SELECT quantity FROM inventories
      WHERE id = ${productId}
      FOR UPDATE  -- 排他ロック
    `;

    if (!inventory[0] || inventory[0].quantity < quantity) {
      throw new ConflictError('Insufficient inventory');
    }

    await tx.inventory.update({
      where: { id: productId },
      data: { quantity: { decrement: quantity } },
    });
  }, {
    timeout: 5000, // 5秒でタイムアウト
  });
}
```

---

## Sagaパターン（分散トランザクション）

```typescript
// Stripe決済 + DB更新のSaga
async function createOrderSaga(input: CreateOrderInput): Promise<Order> {
  let stripePaymentIntentId: string | undefined;

  try {
    // Step 1: 在庫を仮押さえ
    await reserveInventory(input.items);

    // Step 2: Stripe決済作成
    const paymentIntent = await stripe.paymentIntents.create({
      amount: calculateTotal(input.items),
      currency: 'jpy',
    });
    stripePaymentIntentId = paymentIntent.id;

    // Step 3: DB注文作成
    const order = await prisma.order.create({
      data: {
        items: input.items,
        paymentIntentId: paymentIntent.id,
        status: 'pending',
      },
    });

    return order;

  } catch (err) {
    // 補償トランザクション（ロールバック）
    if (stripePaymentIntentId) {
      // Stripe決済をキャンセル
      await stripe.paymentIntents.cancel(stripePaymentIntentId).catch(e =>
        logger.error({ e, stripePaymentIntentId }, 'Failed to cancel payment intent')
      );
    }

    // 在庫仮押さえを解放
    await releaseInventoryReservation(input.items).catch(e =>
      logger.error({ e }, 'Failed to release inventory')
    );

    throw err;
  }
}
```

---

## まとめ

Claude CodeでDBトランザクションを設計する：

1. **CLAUDE.md** に複数テーブル書き込みはトランザクション必須・I/O禁止・ロック戦略を明記
2. **楽観的ロック** でversionカラムによる競合検知（低競合環境向け）
3. **SELECT FOR UPDATE** で行ロック（高競合環境向け）
4. **Sagaパターン** で外部サービスを含む分散トランザクション

---

*トランザクション設計のレビュー（ロック未設定、デッドロックリスク）は **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
