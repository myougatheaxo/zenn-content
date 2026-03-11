---
title: "Claude CodeでCDC・Outboxパターンを設計する：確実なイベント配信・トランザクション整合性"
emoji: "📬"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "kafka"]
published: true
published_at: "2026-03-12 18:00"
---

## はじめに

「DBに保存したのにイベントが飛ばなかった」——Transactional Outboxパターンで書き込みとイベント配信を同一トランザクションにまとめ、確実なイベント配信を実現する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにOutbox/CDC設計ルールを書く

```markdown
## Transactional Outbox設計ルール

### Outboxパターン
- ビジネスロジックとOutbox書き込みは同一トランザクション
- Outboxテーブルは未配信イベントのバッファ
- ポーリング or CDC（debezium）で配信

### 配信保証
- At-least-once配信（重複あり、べき等性で対応）
- 配信成功後にOutboxをマーク（delivered_at）
- 配信失敗は3回リトライ後にDLQ

### CDC（Change Data Capture）
- Debeziumで変更をKafkaに流す
- WALからリアルタイム取得（ポーリング不要）
- Tombstoneレコードで削除イベントを処理
```

---

## Outboxパターンの生成

```
Transactional Outboxパターンを設計してください。

要件：
- 同一トランザクションでのOutbox書き込み
- ポーリングベースの配信ワーカー
- Debezium CDC統合
- べき等性処理

生成ファイル: src/outbox/
```

---

## 生成されるOutboxパターン実装

```typescript
// src/outbox/outboxService.ts

// Outboxテーブル構造
// id, aggregate_type, aggregate_id, event_type, payload, status, created_at, delivered_at, retry_count

// ビジネスロジックとOutbox書き込みを同一トランザクションで
export async function createOrderWithEvent(data: CreateOrderInput): Promise<Order> {
  return prisma.$transaction(async (tx) => {
    // ビジネスロジック
    const order = await tx.order.create({
      data: {
        userId: data.userId,
        items: { createMany: { data: data.items } },
        totalCents: calculateTotal(data.items),
        status: 'pending',
      },
    });

    // 同一トランザクションでOutboxに書き込み
    // DBがコミットされればイベントも確実に配信される
    await tx.outboxEvent.create({
      data: {
        aggregateType: 'Order',
        aggregateId: order.id,
        eventType: 'OrderCreated',
        payload: {
          orderId: order.id,
          userId: data.userId,
          totalCents: order.totalCents,
          items: data.items,
          createdAt: new Date().toISOString(),
        },
        status: 'pending',
      },
    });

    return order;
  });
}

// 注文キャンセル時のイベント
export async function cancelOrderWithEvent(orderId: string, reason: string): Promise<void> {
  await prisma.$transaction(async (tx) => {
    const order = await tx.order.update({
      where: { id: orderId, status: { in: ['pending', 'confirmed'] } },
      data: { status: 'cancelled', cancelledAt: new Date() },
    });

    await tx.outboxEvent.create({
      data: {
        aggregateType: 'Order',
        aggregateId: orderId,
        eventType: 'OrderCancelled',
        payload: { orderId, reason, cancelledAt: new Date().toISOString() },
        status: 'pending',
      },
    });
  });
}
```

```typescript
// src/outbox/outboxWorker.ts — ポーリングベースの配信ワーカー

export class OutboxWorker {
  private running = false;

  async start(): Promise<void> {
    this.running = true;
    while (this.running) {
      await this.processBatch();
      await new Promise(r => setTimeout(r, 1_000)); // 1秒ごとにポーリング
    }
  }

  private async processBatch(): Promise<void> {
    // 未配信イベントをバッチ取得（FOR UPDATEでロック）
    const events = await prisma.$queryRaw<OutboxEvent[]>`
      SELECT * FROM outbox_events
      WHERE status = 'pending' AND retry_count < 3
      ORDER BY created_at ASC
      LIMIT 100
      FOR UPDATE SKIP LOCKED  -- 他のワーカーが処理中のものをスキップ
    `;

    await Promise.allSettled(events.map(event => this.deliverEvent(event)));
  }

  private async deliverEvent(event: OutboxEvent): Promise<void> {
    try {
      // Kafkaに配信
      await kafkaProducer.send({
        topic: `${event.aggregateType.toLowerCase()}-events`,
        messages: [{
          key: event.aggregateId,
          value: JSON.stringify({
            eventId: event.id,
            eventType: event.eventType,
            aggregateId: event.aggregateId,
            payload: event.payload,
            timestamp: event.createdAt.toISOString(),
          }),
        }],
      });

      // 配信成功をマーク
      await prisma.outboxEvent.update({
        where: { id: event.id },
        data: { status: 'delivered', deliveredAt: new Date() },
      });
    } catch (err) {
      // リトライカウントを増加
      await prisma.outboxEvent.update({
        where: { id: event.id },
        data: {
          retryCount: { increment: 1 },
          status: event.retryCount >= 2 ? 'dead_letter' : 'pending',
          lastError: String(err),
        },
      });
      logger.error({ eventId: event.id, err }, 'Failed to deliver outbox event');
    }
  }
}
```

```yaml
# docker-compose.yml — Debezium CDC設定（WALから直接取得）

# debezium/connectors/order-connector.json
```

```json
{
  "name": "order-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.dbname": "myapp",
    "database.server.name": "myapp",
    "table.include.list": "public.outbox_events",
    "plugin.name": "pgoutput",
    "publication.autocreate.mode": "filtered",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.route.topic.replacement": "${routedByValue}-events"
  }
}
```

```typescript
// src/outbox/eventConsumer.ts — Kafkaイベントのべき等消費

export class OrderEventConsumer {
  async handleEvent(event: OutboxMessage): Promise<void> {
    // べき等性チェック（同じeventIdを2回処理しない）
    const processed = await prisma.processedEvent.findUnique({
      where: { eventId: event.eventId },
    });

    if (processed) {
      logger.info({ eventId: event.eventId }, 'Event already processed, skipping');
      return;
    }

    switch (event.eventType) {
      case 'OrderCreated':
        await this.handleOrderCreated(event.payload);
        break;
      case 'OrderCancelled':
        await this.handleOrderCancelled(event.payload);
        break;
    }

    // 処理済みとして記録
    await prisma.processedEvent.create({
      data: { eventId: event.eventId, processedAt: new Date() },
    });
  }
}
```

---

## まとめ

Claude CodeでTransactional Outboxを設計する：

1. **CLAUDE.md** にビジネスロジックとOutbox同一トランザクション・at-least-once・べき等性対応を明記
2. **$transaction内でOutbox書き込み** でDBコミット=イベント配信確実（2フェーズコミット不要）
3. **FOR UPDATE SKIP LOCKED** で複数ワーカーが安全にバッチ処理（競合なし）
4. **Debezium CDC** でWALからリアルタイムにKafkaへ流す（ポーリング遅延なし）

---

*Outboxパターン設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
