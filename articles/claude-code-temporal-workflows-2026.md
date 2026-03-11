---
title: "Claude CodeでTemporal.io耐久性ワークフローを設計する：長時間処理・リトライ・Saga補償"
emoji: "⏳"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "temporal", "microservices"]
published: true
published_at: "2026-03-15 18:00"
---

## はじめに

「注文処理中にサーバーがクラッシュして在庫は引き当て済みなのに支払いが未処理」——Temporal.ioの耐久性ワークフローでSagaパターンを実装し、障害後の自動リカバリを実現する。Claude Codeに設計させる。

---

## CLAUDE.mdにTemporal設計ルールを書く

```markdown
## Temporal.io耐久性ワークフロー設計ルール

### ワークフロー設計
- 1ビジネストランザクション = 1ワークフロー
- アクティビティ: 外部I/O（DB, API, メール）を担当
- ワークフロー: ビジネスロジック・フロー制御のみ（副作用禁止）

### リトライ設定
- アクティビティリトライ: 3回、指数バックオフ（1s, 2s, 4s）
- 冪等アクティビティID: `${workflowId}-${stepName}`
- タイムアウト: scheduleToClose 5分（アクティビティ単位）

### Sagaパターン
- 各ステップで補償アクティビティを登録
- 失敗時は登録済み補償を逆順で実行
- 補償も失敗する可能性 → dead letter notification
```

---

## Temporalワークフローの生成

```
Temporal.ioによる注文処理ワークフローを設計してください。

要件：
- 在庫確認→支払い→発送の3ステップ
- 各ステップの補償（Sagaパターン）
- 失敗時の自動ロールバック
- 冪等処理

生成ファイル: src/temporal/
```

---

## 生成されるTemporal実装

```typescript
// src/temporal/workflows/orderWorkflow.ts — 注文ワークフロー

import { proxyActivities, sleep, ApplicationFailure } from '@temporalio/workflow';
import type * as activities from '../activities/orderActivities';

const {
  reserveInventory,
  releaseInventory,
  chargePayment,
  refundPayment,
  createShipment,
  cancelShipment,
  sendOrderConfirmation,
  sendOrderFailureNotification,
} = proxyActivities<typeof activities>({
  startToCloseTimeout: '5m',
  retry: {
    maximumAttempts: 3,
    initialInterval: '1s',
    backoffCoefficient: 2,
  },
});

export interface OrderWorkflowInput {
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number }>;
  paymentMethodId: string;
  totalAmount: number;
}

// Sagaパターン: 補償アクティビティを逆順で実行
class Saga {
  private compensations: Array<() => Promise<void>> = [];

  addCompensation(fn: () => Promise<void>): void {
    this.compensations.unshift(fn); // 逆順で追加
  }

  async compensate(): Promise<void> {
    for (const compensation of this.compensations) {
      try {
        await compensation();
      } catch (err) {
        // 補償失敗はログに記録して継続（全ての補償を試みる）
        console.error('Compensation failed:', err);
      }
    }
  }
}

export async function orderWorkflow(input: OrderWorkflowInput): Promise<{
  success: boolean;
  shipmentId?: string;
}> {
  const { orderId, userId, items, paymentMethodId, totalAmount } = input;
  const saga = new Saga();

  try {
    // Step 1: 在庫確認・引き当て
    const reservation = await reserveInventory({ orderId, items });
    saga.addCompensation(() =>
      releaseInventory({ reservationId: reservation.reservationId })
    );

    // Step 2: 支払い処理
    const payment = await chargePayment({
      orderId,
      userId,
      paymentMethodId,
      amount: totalAmount,
    });
    saga.addCompensation(() =>
      refundPayment({ chargeId: payment.chargeId, amount: totalAmount })
    );

    // Step 3: 発送手配
    const shipment = await createShipment({ orderId, userId, items });
    saga.addCompensation(() =>
      cancelShipment({ shipmentId: shipment.shipmentId })
    );

    // 完了通知
    await sendOrderConfirmation({ orderId, userId, shipmentId: shipment.shipmentId });

    return { success: true, shipmentId: shipment.shipmentId };
  } catch (err) {
    // 失敗: 補償を逆順で実行
    await saga.compensate();
    await sendOrderFailureNotification({ orderId, userId, error: String(err) });

    // ワークフロー自体はApplicationFailureで終了（再試行しない）
    throw ApplicationFailure.nonRetryable(`Order ${orderId} failed: ${err}`);
  }
}
```

```typescript
// src/temporal/activities/orderActivities.ts — アクティビティ実装

import { activityInfo, heartbeat } from '@temporalio/activity';

// アクティビティは冪等で実装（リトライ安全）
export async function reserveInventory(input: {
  orderId: string;
  items: Array<{ productId: string; quantity: number }>;
}): Promise<{ reservationId: string }> {
  const { orderId, items } = input;

  // 冪等チェック: 既に予約済みか確認
  const existing = await prisma.inventoryReservation.findUnique({
    where: { orderId },
  });
  if (existing) return { reservationId: existing.id }; // 冪等

  // トランザクション内で全在庫を原子的に引き当て
  return prisma.$transaction(async (tx) => {
    for (const item of items) {
      const inventory = await tx.inventory.findUnique({
        where: { productId: item.productId },
      });
      if (!inventory || inventory.available < item.quantity) {
        throw new Error(`Insufficient inventory for product ${item.productId}`);
      }
      await tx.inventory.update({
        where: { productId: item.productId },
        data: { available: { decrement: item.quantity } },
      });
    }

    const reservation = await tx.inventoryReservation.create({
      data: { orderId, items, status: 'RESERVED' },
    });
    return { reservationId: reservation.id };
  });
}

export async function chargePayment(input: {
  orderId: string;
  userId: string;
  paymentMethodId: string;
  amount: number;
}): Promise<{ chargeId: string }> {
  // 冪等キー: orderId を使って重複請求を防止
  const charge = await stripe.paymentIntents.create({
    amount: input.amount,
    currency: 'jpy',
    payment_method: input.paymentMethodId,
    confirm: true,
    idempotency_key: `order-${input.orderId}`, // Stripe冪等キー
    metadata: { orderId: input.orderId, userId: input.userId },
  });

  return { chargeId: charge.id };
}

export async function releaseInventory(input: { reservationId: string }): Promise<void> {
  const reservation = await prisma.inventoryReservation.findUnique({
    where: { id: input.reservationId },
  });
  if (!reservation || reservation.status === 'RELEASED') return; // 冪等

  await prisma.$transaction(async (tx) => {
    for (const item of reservation.items as any[]) {
      await tx.inventory.update({
        where: { productId: item.productId },
        data: { available: { increment: item.quantity } },
      });
    }
    await tx.inventoryReservation.update({
      where: { id: input.reservationId },
      data: { status: 'RELEASED' },
    });
  });
}
```

```typescript
// src/temporal/client.ts — ワークフロー起動・クエリ

import { Client, Connection } from '@temporalio/client';

const connection = await Connection.connect({ address: process.env.TEMPORAL_ADDRESS });
export const temporalClient = new Client({ connection, namespace: 'myapp' });

// 注文ワークフロー起動
export async function startOrderWorkflow(input: OrderWorkflowInput): Promise<string> {
  const handle = await temporalClient.workflow.start(orderWorkflow, {
    taskQueue: 'order-processing',
    workflowId: `order-${input.orderId}`,
    args: [input],
  });
  return handle.workflowId;
}

// ステータスポーリング
export async function getOrderWorkflowStatus(workflowId: string) {
  const handle = temporalClient.workflow.getHandle(workflowId);
  const describe = await handle.describe();
  return {
    status: describe.status.name, // RUNNING, COMPLETED, FAILED, ...
    startTime: describe.startTime,
    closeTime: describe.closeTime,
  };
}
```

---

## まとめ

Claude CodeでTemporal.ioワークフローを設計する：

1. **CLAUDE.md** に1トランザクション=1ワークフロー・ワークフロー内副作用禁止・アクティビティ冪等必須を明記
2. **Sagaパターン** で各ステップ完了後に補償アクティビティを登録——失敗時は逆順で自動補償実行
3. **冪等キー** `order-${orderId}` でStripeの重複請求を防止、DB操作も事前存在チェックで安全
4. **ApplicationFailure.nonRetryable()** でSaga補償後は再試行しない——部分補償済みの二重補償を防止

---

*ワークフロー設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
