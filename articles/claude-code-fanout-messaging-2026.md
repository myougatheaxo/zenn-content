---
title: "Claude CodeでFanoutメッセージングを設計する：Pub/Sub・マルチサブスクライバー・フィルタリング"
emoji: "📡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 15:00"
---

## はじめに

「注文完了イベントを複数のサービスに通知したいが、送信先が増えるたびにコードを変更している」——Fanout（扇形配信）パターンでイベントを1回発行するだけで全購読サービスに届ける設計をClaude Codeに生成させる。

---

## CLAUDE.mdにFanoutメッセージング設計ルールを書く

```markdown
## Fanoutメッセージング設計ルール

### トポロジー
- Publisher → Exchange（ルーティング） → 各Subscriber専用キュー
- SubscriberはExchangeに登録するだけ（PublisherはSubscriberを知らない）
- フィルタリング: Subscriberはイベントタイプとメタデータでフィルタ

### 配信保証
- 少なくとも1回配信（at-least-once）
- Subscriberキューに届いたが処理失敗 → DLQへ
- Subscriber追加/削除はExchangeへの登録/削除のみ（Publisher変更不要）

### バックプレッシャー
- Subscriberキューに上限を設けてPublisherを制御
- 処理が追いつかないSubscriberは独自スケーリング
```

---

## Fanoutメッセージング実装の生成

```
Fanoutメッセージングシステムを設計してください。

要件：
- Exchange経由のFanoutルーティング
- Subscriber登録/解除API
- イベントフィルタリング
- 配信状況の追跡

生成ファイル: src/messaging/fanout/
```

---

## 生成されるFanoutメッセージング実装

```typescript
// src/messaging/fanout/eventExchange.ts — イベントExchange

export interface SubscriberRegistration {
  subscriberId: string;
  queueKey: string;        // Subscriber専用のRedis Streamキー
  eventTypes?: string[];   // フィルタ: 受信するイベントタイプ（undefinedは全て）
  filter?: Record<string, unknown>; // メタデータフィルタ（例: { region: 'ap-northeast-1' }）
  maxQueueSize?: number;   // バックプレッシャー制御
  createdAt: Date;
}

export class EventExchange {
  private readonly registryKey = 'fanout:subscribers';

  // Subscriberを登録
  async subscribe(registration: SubscriberRegistration): Promise<void> {
    await redis.hSet(
      this.registryKey,
      registration.subscriberId,
      JSON.stringify(registration)
    );
    logger.info({ subscriberId: registration.subscriberId }, 'Subscriber registered');
  }

  // Subscriberを解除
  async unsubscribe(subscriberId: string): Promise<void> {
    await redis.hDel(this.registryKey, subscriberId);
    logger.info({ subscriberId }, 'Subscriber unregistered');
  }

  // イベントをFanout配信
  async publish(event: {
    eventId: string;
    eventType: string;
    payload: unknown;
    metadata?: Record<string, unknown>;
  }): Promise<{ deliveredTo: string[]; skipped: string[] }> {
    const subscribers = await this.getAllSubscribers();
    const deliveredTo: string[] = [];
    const skipped: string[] = [];

    const pipeline = redis.pipeline();

    for (const sub of subscribers) {
      // フィルタリング: イベントタイプ
      if (sub.eventTypes && !sub.eventTypes.includes(event.eventType)) {
        skipped.push(sub.subscriberId);
        continue;
      }

      // フィルタリング: メタデータ条件
      if (sub.filter && !this.matchesFilter(event.metadata ?? {}, sub.filter)) {
        skipped.push(sub.subscriberId);
        continue;
      }

      // バックプレッシャー: キューサイズチェック
      if (sub.maxQueueSize) {
        const queueLen = await redis.xLen(sub.queueKey);
        if (queueLen >= sub.maxQueueSize) {
          logger.warn({ subscriberId: sub.subscriberId, queueLen }, 'Subscriber queue full — applying backpressure');
          metrics.fanoutBackpressure.inc({ subscriber: sub.subscriberId });
          skipped.push(sub.subscriberId);
          continue;
        }
      }

      // Subscriber専用キューに配信
      pipeline.xAdd(sub.queueKey, '*', {
        eventId: event.eventId,
        eventType: event.eventType,
        payload: JSON.stringify(event.payload),
        metadata: JSON.stringify(event.metadata ?? {}),
        publishedAt: new Date().toISOString(),
      });

      deliveredTo.push(sub.subscriberId);
    }

    await pipeline.exec();

    logger.info(
      { eventId: event.eventId, eventType: event.eventType, deliveredTo: deliveredTo.length, skipped: skipped.length },
      'Event fanout completed'
    );

    return { deliveredTo, skipped };
  }

  private matchesFilter(metadata: Record<string, unknown>, filter: Record<string, unknown>): boolean {
    return Object.entries(filter).every(([key, value]) => metadata[key] === value);
  }

  private async getAllSubscribers(): Promise<SubscriberRegistration[]> {
    const data = await redis.hGetAll(this.registryKey);
    return Object.values(data).map(v => JSON.parse(v) as SubscriberRegistration);
  }
}
```

```typescript
// src/messaging/fanout/fanoutPublisher.ts — イベント発行

export class FanoutPublisher {
  private readonly exchange = new EventExchange();

  async publishOrderCompleted(order: Order): Promise<void> {
    await this.exchange.publish({
      eventId: ulid(),
      eventType: 'order.completed',
      payload: {
        orderId: order.id,
        userId: order.userId,
        totalAmount: order.totalAmount,
        items: order.items,
      },
      metadata: {
        region: process.env.REGION ?? 'ap-northeast-1',
        tenantId: order.tenantId,
      },
    });
  }
}

// src/messaging/fanout/subscriberWorker.ts — Subscriberワーカー

export class SubscriberWorker {
  async startConsuming(
    subscriberId: string,
    queueKey: string,
    handler: (event: unknown) => Promise<void>
  ): Promise<void> {
    let lastId = '0'; // 最後に読んだメッセージID

    while (true) {
      const messages = await redis.xRead(
        { key: queueKey, id: lastId },
        { COUNT: 10, BLOCK: 5000 }  // 5秒ブロッキング読み取り
      );

      if (!messages || messages.length === 0) continue;

      for (const msg of messages[0].messages) {
        const event = JSON.parse(msg.message.payload as string);

        try {
          await handler(event);
          lastId = msg.id;

          // 処理済みIDを記録（再起動後の重複防止）
          await redis.set(`fanout:cursor:${subscriberId}`, lastId);
        } catch (error) {
          logger.error({ subscriberId, messageId: msg.id, error }, 'Event handling failed');
          // DLQへ
          await redis.xAdd(`fanout:dlq:${subscriberId}`, '*', {
            ...msg.message,
            error: (error as Error).message,
          });
          lastId = msg.id; // エラーでも進める（DLQに移動済み）
        }
      }
    }
  }
}

// 使用例: 各Subscriberを登録
const exchange = new EventExchange();

// 在庫サービス: 注文完了イベントのみ受信
await exchange.subscribe({
  subscriberId: 'inventory-service',
  queueKey: 'fanout:queue:inventory-service',
  eventTypes: ['order.completed', 'order.cancelled'],
  maxQueueSize: 10_000,
  createdAt: new Date(),
});

// 分析サービス: 全イベントを受信（フィルタなし）
await exchange.subscribe({
  subscriberId: 'analytics-service',
  queueKey: 'fanout:queue:analytics-service',
  maxQueueSize: 50_000,
  createdAt: new Date(),
});

// 通知サービス: ap-northeast-1リージョンのみ
await exchange.subscribe({
  subscriberId: 'notification-service',
  queueKey: 'fanout:queue:notification-service',
  eventTypes: ['order.completed'],
  filter: { region: 'ap-northeast-1' },
  maxQueueSize: 5_000,
  createdAt: new Date(),
});
```

---

## まとめ

Claude CodeでFanoutメッセージングを設計する：

1. **CLAUDE.md** にExchange→Subscriber専用キューのトポロジー・SubscriberはExchangeへの登録のみ（Publisher変更不要）・バックプレッシャー上限でPublisherを制御を明記
2. **Subscriber専用キュー（Redis Stream）** でPublisherとSubscriberを完全分離——Subscriberが増えても`exchange.publish()`は変更なし。登録するだけで配信対象になる
3. **メタデータフィルタリング** で`{ region: 'ap-northeast-1' }`など条件付き購読——「このイベントはAPリージョンのサービスだけに届ける」を宣言的に設定
4. **バックプレッシャー（maxQueueSize）** でSubscriberが処理できない量を積まれることを防止——キュー満杯のSubscriberはskipして他は正常配信を継続

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
