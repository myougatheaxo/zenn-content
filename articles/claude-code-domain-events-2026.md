---
title: "Claude Codeでドメインイベントを設計する：集約内イベント・ドメイン層の疎結合・副作用の分離"
emoji: "📢"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 20:00"
---

## はじめに

「注文確定処理の中にメール送信・在庫更新・ポイント付与が混在して保守できない」——ドメインイベントで集約の状態変化を記録し、副作用をイベントハンドラーに分離する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにドメインイベント設計ルールを書く

```markdown
## ドメインイベント設計ルール

### イベントの生成
- 集約のメソッドがドメインイベントを生成（return/throwではなく内部に蓄積）
- イベントは過去形の名前（OrderPlaced, PaymentReceived）
- イベントには何が起きたか（型＋識別子＋必要データ）を含める

### イベントの発行
- アプリケーション層がDBコミット後にドメインイベントを発行
- コミット前に発行しない（ロールバックしたイベントが発行されるのを防ぐ）
- 発行はOutboxパターンと組み合わせてトランザクション保証

### ハンドラーの設計
- 各ハンドラーは1つのイベントに対して1つの副作用を担当
- ハンドラーは失敗してもドメインの整合性に影響しない（副作用の分離）
```

---

## ドメインイベント実装の生成

```
ドメインイベントシステムを設計してください。

要件：
- 集約内でのイベント蓄積
- DBコミット後のイベント発行
- イベントハンドラーの登録と実行
- アウトボックスとの統合

生成ファイル: src/domain/events/
```

---

## 生成されるドメインイベント実装

```typescript
// src/domain/events/domainEvent.ts — ドメインイベント基底

export interface DomainEvent {
  readonly eventId: string;
  readonly occurredAt: Date;
  readonly eventType: string;
}

export abstract class AggregateRoot {
  private _domainEvents: DomainEvent[] = [];

  protected addDomainEvent(event: DomainEvent): void {
    this._domainEvents.push(event);
  }

  // アプリケーション層がDBコミット後に呼ぶ
  collectDomainEvents(): DomainEvent[] {
    const events = [...this._domainEvents];
    this._domainEvents = []; // クリア
    return events;
  }
}
```

```typescript
// src/domain/order/order.ts — 集約ルート（Order）

export class Order extends AggregateRoot {
  private _status: OrderStatus;
  private _items: OrderItem[];
  private _totalAmount: number;

  private constructor(
    private readonly _id: string,
    private readonly _userId: string,
    items: OrderItem[],
    status: OrderStatus
  ) {
    super();
    this._items = items;
    this._status = status;
    this._totalAmount = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  }

  static create(userId: string, items: OrderItem[]): Order {
    const order = new Order(ulid(), userId, items, 'draft');

    // ドメインイベントを内部に蓄積（発行はアプリ層に委ねる）
    order.addDomainEvent({
      eventId: ulid(),
      eventType: 'OrderCreated',
      occurredAt: new Date(),
      orderId: order._id,
      userId,
      totalAmount: order._totalAmount,
    } as OrderCreatedEvent);

    return order;
  }

  confirm(): void {
    if (this._status !== 'pending_payment') {
      throw new DomainError(`Cannot confirm order in status '${this._status}'`);
    }

    this._status = 'confirmed';

    this.addDomainEvent({
      eventId: ulid(),
      eventType: 'OrderConfirmed',
      occurredAt: new Date(),
      orderId: this._id,
      userId: this._userId,
      totalAmount: this._totalAmount,
      items: this._items,
    } as OrderConfirmedEvent);
  }

  cancel(reason: string): void {
    if (['delivered', 'refunded'].includes(this._status)) {
      throw new DomainError(`Cannot cancel order in status '${this._status}'`);
    }

    this._status = 'cancelled';

    this.addDomainEvent({
      eventId: ulid(),
      eventType: 'OrderCancelled',
      occurredAt: new Date(),
      orderId: this._id,
      userId: this._userId,
      reason,
    } as OrderCancelledEvent);
  }

  get id(): string { return this._id; }
  get status(): OrderStatus { return this._status; }
  get totalAmount(): number { return this._totalAmount; }
}

// ドメインイベント型定義
export interface OrderCreatedEvent extends DomainEvent {
  eventType: 'OrderCreated';
  orderId: string;
  userId: string;
  totalAmount: number;
}

export interface OrderConfirmedEvent extends DomainEvent {
  eventType: 'OrderConfirmed';
  orderId: string;
  userId: string;
  totalAmount: number;
  items: OrderItem[];
}

export interface OrderCancelledEvent extends DomainEvent {
  eventType: 'OrderCancelled';
  orderId: string;
  userId: string;
  reason: string;
}
```

```typescript
// src/application/eventDispatcher.ts — イベントディスパッチャー

type EventHandler<T extends DomainEvent> = (event: T) => Promise<void>;

export class DomainEventDispatcher {
  private readonly handlers = new Map<string, EventHandler<DomainEvent>[]>();

  on<T extends DomainEvent>(eventType: string, handler: EventHandler<T>): void {
    const existing = this.handlers.get(eventType) ?? [];
    existing.push(handler as EventHandler<DomainEvent>);
    this.handlers.set(eventType, existing);
  }

  async dispatch(events: DomainEvent[]): Promise<void> {
    for (const event of events) {
      const handlers = this.handlers.get(event.eventType) ?? [];

      // ハンドラーを並列実行（独立した副作用）
      await Promise.allSettled(
        handlers.map(handler =>
          handler(event).catch(err =>
            logger.error({ eventType: event.eventType, eventId: event.eventId, err }, 'Event handler failed')
          )
        )
      );
    }
  }
}

// ハンドラー登録
const dispatcher = new DomainEventDispatcher();

// OrderConfirmedEvent ハンドラー群
dispatcher.on<OrderConfirmedEvent>('OrderConfirmed', async (event) => {
  await emailService.sendOrderConfirmation(event.userId, event.orderId);
});

dispatcher.on<OrderConfirmedEvent>('OrderConfirmed', async (event) => {
  await inventoryService.reserveItems(event.orderId, event.items);
});

dispatcher.on<OrderConfirmedEvent>('OrderConfirmed', async (event) => {
  await pointsService.awardPoints(event.userId, event.totalAmount);
});

// アプリケーション層: DB保存 → イベント発行
export class ConfirmOrderUseCase {
  async execute(orderId: string): Promise<void> {
    const order = await orderRepository.findById(orderId);

    order.confirm();  // ドメインイベントを内部に蓄積

    // DBにコミット
    await orderRepository.save(order);

    // コミット後にイベントを発行（トランザクション外）
    const events = order.collectDomainEvents();
    await dispatcher.dispatch(events);  // メール・在庫・ポイントの副作用を分離実行
  }
}
```

---

## まとめ

Claude Codeでドメインイベントを設計する：

1. **CLAUDE.md** に集約メソッドがイベントを内部蓄積・アプリ層がDBコミット後にcollectして発行・ハンドラー失敗はドメインに影響しないを明記
2. **AggregateRoot基底クラス** でイベント蓄積/収集のライフサイクルを統一——`order.confirm()` → `order.collectDomainEvents()` → `dispatcher.dispatch(events)` という一貫したフロー
3. **DBコミット後に発行** することでトランザクションロールバック後にイベントが流れることを防止——「DBは失敗したのにメールだけ送った」問題を解消
4. **ハンドラーの分離（1イベント × 複数ハンドラー）** でOrderConfirmedに対してメール/在庫/ポイントを独立したハンドラーで実装——新しい副作用の追加が`dispatcher.on()`の1行で済む

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
