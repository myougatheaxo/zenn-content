---
title: "Claude Codeでイベント駆動ステートマシンを設計する：注文ワークフロー・状態遷移の型安全な管理"
emoji: "⚙️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-16 21:00"
---

## はじめに

「注文ステータスが不正な遷移になっている」「どのイベントがどの状態遷移を引き起こすか追えない」——TypeScriptの型システムを使ったステートマシンで状態遷移を安全に管理し、不正遷移をコンパイル時に検出する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにステートマシン設計ルールを書く

```markdown
## イベント駆動ステートマシン設計ルール

### 状態定義
- 全ての状態をUnion型で定義（'pending' | 'confirmed' | 'shipped' | ...）
- 遷移マップで許可された遷移のみを定義（それ以外は型エラー）
- 状態変更はDB更新とイベント発行をアトミックに（トランザクション）

### イベント設計
- イベントは不変（実行済みイベントの削除禁止）
- イベントログから現在の状態を再構築可能にする（Event Sourcing）
- イベントにはtimestamp, actorId（誰が操作したか）を含める

### ガード条件
- 遷移前に前提条件を検証（在庫確認・残高確認等）
- ガード失敗時はドメインエラーを返す（状態は変更しない）
```

---

## ステートマシン実装の生成

```
イベント駆動ステートマシンを設計してください。

要件：
- 型安全な状態遷移（TypeScript）
- ガード条件
- イベントログ
- 不正遷移の防止

生成ファイル: src/statemachine/
```

---

## 生成されるステートマシン実装

```typescript
// src/statemachine/orderStateMachine.ts — 注文ステートマシン

// 状態の定義
export type OrderState =
  | 'draft'
  | 'pending_payment'
  | 'confirmed'
  | 'preparing'
  | 'shipped'
  | 'delivered'
  | 'cancelled'
  | 'refunded';

// イベントの定義
export type OrderEvent =
  | { type: 'SUBMIT';         actorId: string }
  | { type: 'PAYMENT_RECEIVED'; actorId: string; paymentId: string }
  | { type: 'PAYMENT_FAILED';   actorId: string; reason: string }
  | { type: 'CONFIRM';          actorId: string }
  | { type: 'START_PREPARING';  actorId: string }
  | { type: 'SHIP';             actorId: string; trackingNumber: string }
  | { type: 'DELIVER';          actorId: string }
  | { type: 'CANCEL';           actorId: string; reason: string }
  | { type: 'REFUND';           actorId: string; amount: number };

// 遷移マップ: { [fromState]: { [eventType]: toState } }
type TransitionMap = {
  [S in OrderState]?: {
    [E in OrderEvent['type']]?: OrderState;
  };
};

const TRANSITIONS: TransitionMap = {
  draft:           { SUBMIT: 'pending_payment' },
  pending_payment: { PAYMENT_RECEIVED: 'confirmed', PAYMENT_FAILED: 'cancelled', CANCEL: 'cancelled' },
  confirmed:       { CONFIRM: 'preparing', CANCEL: 'cancelled' },
  preparing:       { START_PREPARING: 'preparing', SHIP: 'shipped', CANCEL: 'cancelled' },
  shipped:         { DELIVER: 'delivered' },
  delivered:       { REFUND: 'refunded' },
  confirmed:       { CANCEL: 'cancelled' },
};

export class OrderStateMachine {
  private readonly transitions = TRANSITIONS;

  // 遷移が有効かチェック
  canTransition(currentState: OrderState, event: OrderEvent): boolean {
    return !!this.transitions[currentState]?.[event.type];
  }

  // 遷移後の状態を取得（型安全）
  getNextState(currentState: OrderState, event: OrderEvent): OrderState | null {
    return this.transitions[currentState]?.[event.type] ?? null;
  }
}
```

```typescript
// src/statemachine/orderService.ts — ステートマシンを使ったサービス

export class OrderStateMachineService {
  private readonly machine = new OrderStateMachine();

  async transition(
    orderId: string,
    event: OrderEvent,
    guards?: { before?: () => Promise<void> }
  ): Promise<Order> {
    return prisma.$transaction(async (tx) => {
      // 現在の注文を取得（FOR UPDATE で楽観的ロック）
      const order = await tx.$queryRaw<Order[]>`
        SELECT * FROM orders WHERE id = ${orderId} FOR UPDATE
      `.then(rows => rows[0]);

      if (!order) throw new NotFoundError(`Order ${orderId} not found`);

      const currentState = order.status as OrderState;

      // 遷移可否チェック
      if (!this.machine.canTransition(currentState, event)) {
        throw new InvalidStateTransitionError(
          `Cannot apply '${event.type}' to order in state '${currentState}'`
        );
      }

      // ガード条件（外部チェック）
      if (guards?.before) {
        await guards.before(); // 失敗するとエラーがスローされ状態変更なし
      }

      const nextState = this.machine.getNextState(currentState, event)!;

      // イベントをログに記録（Event Sourcing）
      await tx.orderEvent.create({
        data: {
          orderId,
          eventType: event.type,
          fromState: currentState,
          toState: nextState,
          actorId: event.actorId,
          metadata: JSON.stringify(event),
          occurredAt: new Date(),
        },
      });

      // 状態更新
      const updatedOrder = await tx.order.update({
        where: { id: orderId },
        data: {
          status: nextState,
          ...this.extractOrderFields(event, nextState),
        },
      });

      logger.info({ orderId, fromState: currentState, toState: nextState, eventType: event.type }, 'Order state transitioned');

      return updatedOrder;
    });
  }

  private extractOrderFields(event: OrderEvent, nextState: OrderState): Partial<any> {
    switch (event.type) {
      case 'SHIP':          return { trackingNumber: event.trackingNumber, shippedAt: new Date() };
      case 'DELIVER':       return { deliveredAt: new Date() };
      case 'CANCEL':        return { cancelledAt: new Date(), cancelReason: event.reason };
      case 'REFUND':        return { refundedAt: new Date(), refundAmount: event.amount };
      default:              return {};
    }
  }

  // イベントログから状態を再構築（Event Sourcing）
  async rebuildState(orderId: string): Promise<OrderState> {
    const events = await prisma.orderEvent.findMany({
      where: { orderId },
      orderBy: { occurredAt: 'asc' },
    });

    if (events.length === 0) return 'draft';

    return events[events.length - 1].toState as OrderState;
  }
}

// 使用例: 在庫チェック付きの注文確認
const service = new OrderStateMachineService();

await service.transition(orderId, { type: 'CONFIRM', actorId: 'admin-1' }, {
  before: async () => {
    // ガード: 在庫確認
    const allInStock = await inventoryService.checkAllItems(orderId);
    if (!allInStock) {
      throw new InsufficientInventoryError('Cannot confirm: some items are out of stock');
    }
  },
});

// 不正遷移の例（TypeScriptで型エラー + ランタイムエラー）
await service.transition(orderId, { type: 'SHIP', actorId: 'admin-1', trackingNumber: 'XX123' });
// → 'draft'状態のままSHIPしようとすると: InvalidStateTransitionError
```

---

## まとめ

Claude Codeでイベント駆動ステートマシンを設計する：

1. **CLAUDE.md** に全状態をUnion型で定義・遷移マップで許可遷移のみ定義・状態変更とイベントログをトランザクションで原子実行を明記
2. **TRANSITIONS遷移マップ** でどの状態からどのイベントでどの状態へ遷移できるかを宣言的に定義——コードを見れば全遷移が一目瞭然
3. **ガード条件** で在庫確認・残高確認などを遷移前に検証——ガード失敗時はエラーをスローして状態を変更しない
4. **イベントログ（Event Sourcing）** で全遷移を記録——`rebuildState`で任意の時点の状態を再構築でき、デバッグ・監査ログとして活用

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
