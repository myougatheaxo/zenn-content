---
title: "Claude CodeでSagaパターンを設計する：分散トランザクション・補償トランザクション・オーケストレーション"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 09:00"
---

## はじめに

「マイクロサービスをまたぐ注文処理が途中で失敗した——どこまで戻せばいい？」——Sagaパターンで分散トランザクションを補償トランザクションで巻き戻し、整合性を保つ設計をClaude Codeに生成させる。

---

## CLAUDE.mdにSagaパターン設計ルールを書く

```markdown
## Sagaパターン設計ルール

### オーケストレーション型
- SagaオーケストレーターがステップをRedisキューで順番に実行
- 各ステップは成功/失敗/補償の3つのハンドラーを持つ
- 失敗時は逆順に補償トランザクションを実行

### 冪等性
- 各ステップはリトライ可能（冪等キー必須）
- 補償トランザクションも冪等（二重実行で同じ結果）
- SagaログをDBに永続化（クラッシュ後の再開）

### タイムアウト
- 各ステップに最大待機時間（例: 30秒）
- タイムアウト時は自動補償開始
- SagaID単位でステータスを追跡
```

---

## Saga実装の生成

```
Sagaパターンを設計してください。

要件：
- オーケストレーション型Saga
- 補償トランザクション
- 冪等なステップ実行
- Sagaログ永続化

生成ファイル: src/saga/
```

---

## 生成されるSaga実装

```typescript
// src/saga/sagaOrchestrator.ts — Sagaオーケストレーター

export interface SagaStep<TContext> {
  name: string;
  execute: (context: TContext, sagaId: string) => Promise<TContext>;
  compensate: (context: TContext, sagaId: string) => Promise<void>;
  timeoutMs?: number;
}

export interface SagaLog {
  sagaId: string;
  stepName: string;
  status: 'executing' | 'completed' | 'compensating' | 'compensated' | 'failed';
  context: unknown;
  executedAt: Date;
  completedAt?: Date;
  error?: string;
}

export class SagaOrchestrator<TContext extends Record<string, unknown>> {
  private readonly steps: SagaStep<TContext>[];

  constructor(steps: SagaStep<TContext>[]) {
    this.steps = steps;
  }

  async execute(initialContext: TContext): Promise<{ sagaId: string; finalContext: TContext }> {
    const sagaId = ulid();
    let context = { ...initialContext };
    const completedSteps: number[] = [];

    logger.info({ sagaId, steps: this.steps.map(s => s.name) }, 'Saga started');

    for (let i = 0; i < this.steps.length; i++) {
      const step = this.steps[i];

      // Sagaログに記録
      await this.logStep(sagaId, step.name, 'executing', context);

      try {
        // タイムアウト付きで実行
        context = await this.withTimeout(
          step.execute(context, sagaId),
          step.timeoutMs ?? 30_000,
          `Step '${step.name}' timed out`
        );

        await this.logStep(sagaId, step.name, 'completed', context);
        completedSteps.push(i);

        logger.info({ sagaId, step: step.name }, 'Saga step completed');
      } catch (error) {
        logger.error({ sagaId, step: step.name, error }, 'Saga step failed, starting compensation');

        // 逆順に補償トランザクションを実行
        await this.compensate(sagaId, context, completedSteps);

        throw new SagaExecutionError(sagaId, step.name, error as Error);
      }
    }

    logger.info({ sagaId }, 'Saga completed successfully');
    return { sagaId, finalContext: context };
  }

  private async compensate(
    sagaId: string,
    context: TContext,
    completedSteps: number[]
  ): Promise<void> {
    // 逆順に補償
    for (const stepIndex of [...completedSteps].reverse()) {
      const step = this.steps[stepIndex];

      await this.logStep(sagaId, step.name, 'compensating', context);

      try {
        await step.compensate(context, sagaId);
        await this.logStep(sagaId, step.name, 'compensated', context);
        logger.info({ sagaId, step: step.name }, 'Compensation completed');
      } catch (compensateError) {
        // 補償失敗は致命的 — アラートしてスキップ
        logger.error({ sagaId, step: step.name, error: compensateError }, 'COMPENSATION FAILED — manual intervention required');
        await this.logStep(sagaId, step.name, 'failed', context, (compensateError as Error).message);
      }
    }
  }

  private async withTimeout<T>(promise: Promise<T>, ms: number, message: string): Promise<T> {
    const timeout = new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(message)), ms)
    );
    return Promise.race([promise, timeout]);
  }

  private async logStep(
    sagaId: string,
    stepName: string,
    status: SagaLog['status'],
    context: TContext,
    error?: string
  ): Promise<void> {
    await prisma.sagaLog.upsert({
      where: { sagaId_stepName: { sagaId, stepName } },
      create: { sagaId, stepName, status, context, error, executedAt: new Date() },
      update: { status, error, completedAt: new Date() },
    });
  }
}
```

```typescript
// src/saga/orderSaga.ts — 注文処理Saga

interface OrderSagaContext {
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  paymentId?: string;
  reservationId?: string;
  totalAmount: number;
}

// Sagaステップ定義
const ORDER_SAGA_STEPS: SagaStep<OrderSagaContext>[] = [
  {
    name: 'reserve-inventory',
    execute: async (ctx, sagaId) => {
      // 冪等キー = sagaId でリトライ安全
      const reservation = await inventoryService.reserve({
        idempotencyKey: `${sagaId}:reserve`,
        items: ctx.items,
      });
      return { ...ctx, reservationId: reservation.id };
    },
    compensate: async (ctx, sagaId) => {
      if (!ctx.reservationId) return;
      await inventoryService.cancelReservation({
        idempotencyKey: `${sagaId}:cancel-reserve`,
        reservationId: ctx.reservationId,
      });
    },
    timeoutMs: 10_000,
  },
  {
    name: 'charge-payment',
    execute: async (ctx, sagaId) => {
      const payment = await paymentService.charge({
        idempotencyKey: `${sagaId}:charge`,
        userId: ctx.userId,
        amount: ctx.totalAmount,
        orderId: ctx.orderId,
      });
      return { ...ctx, paymentId: payment.id };
    },
    compensate: async (ctx, sagaId) => {
      if (!ctx.paymentId) return;
      await paymentService.refund({
        idempotencyKey: `${sagaId}:refund`,
        paymentId: ctx.paymentId,
        reason: 'saga-compensation',
      });
    },
    timeoutMs: 15_000,
  },
  {
    name: 'confirm-order',
    execute: async (ctx) => {
      await prisma.order.update({
        where: { id: ctx.orderId },
        data: { status: 'confirmed', paymentId: ctx.paymentId },
      });
      return ctx;
    },
    compensate: async (ctx) => {
      await prisma.order.update({
        where: { id: ctx.orderId },
        data: { status: 'cancelled', cancelledAt: new Date() },
      });
    },
  },
];

// 使用例
export async function processOrderWithSaga(orderId: string): Promise<void> {
  const order = await prisma.order.findUniqueOrThrow({ where: { id: orderId }, include: { items: true } });

  const saga = new SagaOrchestrator<OrderSagaContext>(ORDER_SAGA_STEPS);

  try {
    const { finalContext } = await saga.execute({
      orderId,
      userId: order.userId,
      items: order.items.map(i => ({ productId: i.productId, quantity: i.quantity, price: i.price })),
      totalAmount: order.totalAmount,
    });

    logger.info({ orderId, paymentId: finalContext.paymentId }, 'Order saga succeeded');
  } catch (error) {
    if (error instanceof SagaExecutionError) {
      logger.error({ orderId, sagaId: error.sagaId, failedStep: error.stepName }, 'Order saga failed, compensation executed');
      // 補償完了済み — UI/メールに失敗を通知
    }
    throw error;
  }
}
```

---

## まとめ

Claude CodeでSagaパターンを設計する：

1. **CLAUDE.md** にオーケストレーション型・各ステップに成功/補償ハンドラー・Sagaログ永続化・タイムアウトを明記
2. **逆順補償** で`completedSteps`を逆順にループ——「在庫予約→決済→注文確定」が失敗した場合、「注文キャンセル→返金→予約解除」の順で巻き戻し
3. **冪等キー（sagaId:stepName）** でリトライを安全に——ネットワーク障害でリトライされても二重課金・二重予約を防止
4. **補償失敗はアラート** ——補償自体が失敗した場合は自動復旧を諦めてオペレーターに通知（補償の補償は設計しない）

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
