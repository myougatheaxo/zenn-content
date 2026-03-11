---
title: "Claude CodeでEvent Sourcingを設計する：イベントストア・プロジェクション・CQRS"
emoji: "📜"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
---

## はじめに

「注文がなぜキャンセルされたか」を後から調べたい——通常のCRUDでは最終状態しか残らない。Event Sourcingで全変更履歴をイベントとして保存し、任意の時点の状態を再現できるようにする。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにEvent Sourcing設計ルールを書く

```markdown
## Event Sourcing設計ルール

### イベント設計
- イベントは過去形で命名（OrderPlaced、OrderCancelled）
- イベントは変更不可（append-only）
- イベントにはaggregateId、version、occurredAtを含める
- ペイロードはJSON（スキーマバージョニング必須）

### アグリゲート
- 状態はイベントの再生で導出（stateを直接変更しない）
- バージョン番号で楽観的ロック（競合検知）
- 1アグリゲート = 1トランザクション（他のアグリゲートと同時更新しない）

### プロジェクション（Read Model）
- イベントから読み取り用データを非同期で構築
- プロジェクションは削除・再構築可能（イベントが正）
- CQRSで書き込みモデルと読み取りモデルを分離
```

---

## Event Sourcingシステムの生成

```
注文管理のEvent Sourcingシステムを設計してください。

イベント: OrderPlaced, ItemAdded, OrderCancelled
要件：
- PostgreSQLのイベントストア
- アグリゲートの再構築
- 楽観的ロック
- プロジェクション（READ用）

生成ファイル: src/eventSourcing/
```

---

## 生成されるEvent Sourcing実装

```typescript
// src/eventSourcing/events.ts

// イベント型定義
type OrderEvent =
  | { type: 'OrderPlaced'; data: { userId: string; items: OrderItem[] } }
  | { type: 'ItemAdded'; data: { item: OrderItem } }
  | { type: 'OrderCancelled'; data: { reason: string; cancelledBy: string } }
  | { type: 'OrderShipped'; data: { trackingNumber: string; carrier: string } };

interface StoredEvent {
  id: string;
  aggregateId: string;
  aggregateType: string;
  version: number;
  type: string;
  data: unknown;
  occurredAt: Date;
}
```

```typescript
// src/eventSourcing/eventStore.ts

export class EventStore {
  // イベントを保存（楽観的ロック）
  async append(
    aggregateId: string,
    aggregateType: string,
    events: OrderEvent[],
    expectedVersion: number
  ): Promise<void> {
    await prisma.$transaction(async (tx) => {
      // 現在のバージョンを取得してロック確認
      const currentVersion = await tx.$queryRaw<Array<{ max: number }>>`
        SELECT COALESCE(MAX(version), 0) as max
        FROM events
        WHERE aggregate_id = ${aggregateId}
        FOR UPDATE  -- 同一aggregateの並行書き込みを防ぐ
      `;

      const latestVersion = currentVersion[0].max;

      if (latestVersion !== expectedVersion) {
        throw new OptimisticLockError(
          `Expected version ${expectedVersion}, got ${latestVersion}`
        );
      }

      // イベントをバルクインサート
      let version = expectedVersion;
      for (const event of events) {
        version++;
        await tx.event.create({
          data: {
            aggregateId,
            aggregateType,
            version,
            type: event.type,
            data: event.data,
            occurredAt: new Date(),
          },
        });
      }
    });
  }

  // アグリゲートの全イベントを取得
  async getEvents(aggregateId: string, fromVersion = 0): Promise<StoredEvent[]> {
    return prisma.event.findMany({
      where: {
        aggregateId,
        version: { gt: fromVersion },
      },
      orderBy: { version: 'asc' },
    });
  }
}

export const eventStore = new EventStore();
```

```typescript
// src/eventSourcing/orderAggregate.ts

interface OrderState {
  id: string;
  userId: string;
  items: OrderItem[];
  status: 'pending' | 'confirmed' | 'shipped' | 'cancelled';
  version: number;
}

export class OrderAggregate {
  private state: OrderState;
  private pendingEvents: OrderEvent[] = [];

  private constructor(state: OrderState) {
    this.state = state;
  }

  // イベントから状態を再構築（スナップショットから再開可能）
  static fromEvents(aggregateId: string, events: StoredEvent[]): OrderAggregate {
    const initial: OrderState = {
      id: aggregateId,
      userId: '',
      items: [],
      status: 'pending',
      version: 0,
    };

    const state = events.reduce((acc, event) => {
      return OrderAggregate.applyEvent(acc, event);
    }, initial);

    return new OrderAggregate(state);
  }

  // イベントを適用して状態を更新
  private static applyEvent(state: OrderState, event: StoredEvent): OrderState {
    switch (event.type) {
      case 'OrderPlaced': {
        const data = event.data as { userId: string; items: OrderItem[] };
        return { ...state, userId: data.userId, items: data.items, version: event.version };
      }
      case 'ItemAdded': {
        const data = event.data as { item: OrderItem };
        return { ...state, items: [...state.items, data.item], version: event.version };
      }
      case 'OrderCancelled': {
        return { ...state, status: 'cancelled', version: event.version };
      }
      case 'OrderShipped': {
        return { ...state, status: 'shipped', version: event.version };
      }
      default:
        return state;
    }
  }

  // コマンド: 注文作成
  static place(orderId: string, userId: string, items: OrderItem[]): OrderAggregate {
    const aggregate = new OrderAggregate({
      id: orderId, userId, items, status: 'pending', version: 0,
    });
    aggregate.pendingEvents.push({ type: 'OrderPlaced', data: { userId, items } });
    return aggregate;
  }

  // コマンド: 注文キャンセル
  cancel(reason: string, cancelledBy: string): void {
    if (this.state.status !== 'pending') {
      throw new BusinessError(`Cannot cancel order in status: ${this.state.status}`);
    }
    this.pendingEvents.push({ type: 'OrderCancelled', data: { reason, cancelledBy } });
    this.state = { ...this.state, status: 'cancelled' };
  }

  // 保存してイベントをクリア
  async save(): Promise<void> {
    await eventStore.append(
      this.state.id,
      'Order',
      this.pendingEvents,
      this.state.version
    );
    this.pendingEvents = [];
  }

  getState(): Readonly<OrderState> {
    return this.state;
  }
}
```

---

## プロジェクション（READ用のDenormalized View）

```typescript
// src/eventSourcing/projections/orderProjection.ts
// イベントをポーリングして読み取り用DBを更新

export async function rebuildOrderProjection(): Promise<void> {
  // 最後に処理したイベントIDを取得
  const checkpoint = await prisma.projectionCheckpoint.findUnique({
    where: { name: 'order_projection' },
  });

  const lastProcessedId = checkpoint?.lastEventId ?? 0;

  // 未処理イベントをバッチで取得
  const events = await prisma.event.findMany({
    where: {
      aggregateType: 'Order',
      id: { gt: lastProcessedId },
    },
    orderBy: { id: 'asc' },
    take: 100,
  });

  for (const event of events) {
    await processOrderEvent(event);
  }

  // チェックポイント更新
  if (events.length > 0) {
    await prisma.projectionCheckpoint.upsert({
      where: { name: 'order_projection' },
      create: { name: 'order_projection', lastEventId: events[events.length - 1].id },
      update: { lastEventId: events[events.length - 1].id },
    });
  }
}

async function processOrderEvent(event: StoredEvent): Promise<void> {
  switch (event.type) {
    case 'OrderPlaced': {
      const data = event.data as { userId: string; items: OrderItem[] };
      await prisma.orderView.create({
        data: {
          id: event.aggregateId,
          userId: data.userId,
          status: 'pending',
          total: calculateTotal(data.items),
          itemCount: data.items.length,
        },
      });
      break;
    }
    case 'OrderCancelled': {
      await prisma.orderView.update({
        where: { id: event.aggregateId },
        data: { status: 'cancelled' },
      });
      break;
    }
  }
}
```

---

## まとめ

Claude CodeでEvent Sourcingを設計する：

1. **CLAUDE.md** にイベント過去形命名・append-only・アグリゲート境界を明記
2. **楽観的ロック** でversionチェック（同一aggregateの並行書き込みを防ぐ）
3. **applyEvent** でイベントを適用して状態を再構築（任意の時点に戻れる）
4. **プロジェクション** でイベントからREAD用のDenormalized Viewを非同期構築

---

*Event Sourcing設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
