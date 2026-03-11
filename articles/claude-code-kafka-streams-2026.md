---
title: "Claude CodeでKafkaストリーム処理を設計する：KafkaJS・コンシューマーグループ・DLQ"
emoji: "📨"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "kafka", "redis"]
published: true
published_at: "2026-03-13 10:00"
---

## はじめに

メッセージキューとストリーム処理の違いを理解しながら——KafkaJSでコンシューマーグループ・オフセット管理・デッドレターキューを設計する。Claude Codeに生成させる。

---

## CLAUDE.mdにKafka設計ルールを書く

```markdown
## Kafkaストリーム処理設計ルール

### トピック設計
- 1イベントタイプ = 1トピック原則
- パーティション数: throughput目標 / 単一コンシューマースループット
- リテンション: 7日（重要イベントは30日）
- コンパクション: 最新状態スナップショット用

### コンシューマー設計
- autoCommit: false（手動オフセット管理）
- エラー時はリトライ3回 → DLQトピックへ
- 冪等性: processingIdで重複処理検知
- graceful shutdown: consumer.disconnect()を確実に

### プロデューサー設計
- acks: 'all'（全レプリカへの書き込み確認）
- idempotent: true（プロデューサー冪等性）
- compression: CompressionTypes.GZIP
```

---

## Kafkaストリーム処理の生成

```
KafkaJS + TypeScriptでイベントストリーム処理を設計してください。

要件：
- コンシューマーグループ
- 手動オフセット管理
- エラー時DLQ
- 冪等コンシューマー
- グレースフルシャットダウン

生成ファイル: src/kafka/
```

---

## 生成されるKafka実装

```typescript
// src/kafka/producer.ts — プロデューサー

import { Kafka, CompressionTypes, Partitioners } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'myapp-producer',
  brokers: process.env.KAFKA_BROKERS!.split(','),
  ssl: process.env.NODE_ENV === 'production',
  sasl: process.env.KAFKA_USERNAME ? {
    mechanism: 'scram-sha-256',
    username: process.env.KAFKA_USERNAME,
    password: process.env.KAFKA_PASSWORD!,
  } : undefined,
});

export const producer = kafka.producer({
  idempotent: true,  // 冪等プロデューサー（acks: all自動）
  createPartitioner: Partitioners.DefaultPartitioner,
});

// 型安全なイベント送信
interface KafkaEvent<T = unknown> {
  eventId: string;   // UUID（冪等性追跡）
  eventType: string;
  occurredAt: string; // ISO8601
  payload: T;
}

export async function publishEvent<T>(
  topic: string,
  payload: T,
  key?: string
): Promise<void> {
  const event: KafkaEvent<T> = {
    eventId: crypto.randomUUID(),
    eventType: topic,
    occurredAt: new Date().toISOString(),
    payload,
  };

  await producer.send({
    topic,
    compression: CompressionTypes.GZIP,
    messages: [{
      key: key ?? null,
      value: JSON.stringify(event),
      headers: {
        'event-type': topic,
        'event-id': event.eventId,
        'content-type': 'application/json',
      },
    }],
  });

  logger.debug({ topic, eventId: event.eventId }, 'Event published');
}
```

```typescript
// src/kafka/consumer.ts — コンシューマー基底クラス

export abstract class BaseKafkaConsumer<T = unknown> {
  private consumer: Consumer;
  private isRunning = false;

  constructor(
    private readonly topic: string,
    private readonly groupId: string,
  ) {
    this.consumer = kafka.consumer({
      groupId,
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
    });
  }

  abstract process(event: KafkaEvent<T>, partition: number, offset: string): Promise<void>;

  async start(): Promise<void> {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: this.topic, fromBeginning: false });

    this.isRunning = true;

    await this.consumer.run({
      autoCommit: false, // 手動オフセット管理
      eachMessage: async ({ topic, partition, message, heartbeat }) => {
        const rawValue = message.value?.toString();
        if (!rawValue) return;

        let event: KafkaEvent<T>;
        try {
          event = JSON.parse(rawValue);
        } catch (e) {
          logger.error({ rawValue }, 'Failed to parse Kafka message');
          await this.sendToDLQ(message, 'PARSE_ERROR', String(e));
          await this.commitOffset(partition, message.offset);
          return;
        }

        // 冪等性チェック（重複処理防止）
        const alreadyProcessed = await this.checkProcessed(event.eventId);
        if (alreadyProcessed) {
          logger.debug({ eventId: event.eventId }, 'Duplicate event skipped');
          await this.commitOffset(partition, message.offset);
          return;
        }

        // リトライ付き処理
        let lastError: Error | null = null;
        for (let attempt = 1; attempt <= 3; attempt++) {
          try {
            await this.process(event, partition, message.offset);
            await this.markProcessed(event.eventId);
            await this.commitOffset(partition, message.offset);
            await heartbeat(); // タイムアウト防止
            return;
          } catch (err) {
            lastError = err as Error;
            logger.warn({ eventId: event.eventId, attempt, error: lastError.message }, 'Processing failed, retrying');
            if (attempt < 3) await sleep(1000 * attempt); // 指数バックオフ
          }
        }

        // 3回失敗 → DLQ
        await this.sendToDLQ(message, 'PROCESSING_FAILED', lastError!.message);
        await this.commitOffset(partition, message.offset);
        logger.error({ eventId: event.eventId }, 'Event sent to DLQ after 3 failures');
      },
    });
  }

  private async commitOffset(partition: number, offset: string): Promise<void> {
    await this.consumer.commitOffsets([{
      topic: this.topic,
      partition,
      offset: (BigInt(offset) + 1n).toString(), // 次のオフセットをコミット
    }]);
  }

  private async sendToDLQ(
    originalMessage: KafkaMessage,
    reason: string,
    error: string
  ): Promise<void> {
    await producer.send({
      topic: `${this.topic}.dlq`,
      messages: [{
        value: originalMessage.value,
        headers: {
          ...originalMessage.headers,
          'dlq-reason': reason,
          'dlq-error': error,
          'dlq-source-topic': this.topic,
          'dlq-group-id': this.groupId,
          'dlq-timestamp': new Date().toISOString(),
        },
      }],
    });
  }

  private async checkProcessed(eventId: string): Promise<boolean> {
    return !!(await redis.get(`processed:${eventId}`));
  }

  private async markProcessed(eventId: string): Promise<void> {
    await redis.set(`processed:${eventId}`, '1', { EX: 86400 }); // 24時間保持
  }

  async stop(): Promise<void> {
    this.isRunning = false;
    await this.consumer.disconnect();
    logger.info({ topic: this.topic }, 'Kafka consumer disconnected');
  }
}
```

```typescript
// src/kafka/consumers/orderConsumer.ts — 注文イベントコンシューマー

interface OrderCreatedPayload {
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  totalAmount: number;
}

export class OrderCreatedConsumer extends BaseKafkaConsumer<OrderCreatedPayload> {
  constructor() {
    super('order.created', 'order-processor');
  }

  async process(event: KafkaEvent<OrderCreatedPayload>): Promise<void> {
    const { orderId, userId, items, totalAmount } = event.payload;

    // 注文処理をDBトランザクション内で実行
    await prisma.$transaction(async (tx) => {
      // 在庫を減算
      for (const item of items) {
        await tx.productInventory.update({
          where: { productId: item.productId },
          data: { quantity: { decrement: item.quantity } },
        });
      }

      // 注文ステータスを更新
      await tx.order.update({
        where: { id: orderId },
        data: { status: 'CONFIRMED', processedAt: new Date() },
      });
    });

    // 確認メール送信イベントを発行
    await publishEvent('notification.send', {
      userId,
      type: 'ORDER_CONFIRMED',
      data: { orderId, totalAmount },
    });

    logger.info({ orderId, userId }, 'Order processed successfully');
  }
}

// src/kafka/index.ts — グレースフルシャットダウン
const consumers: BaseKafkaConsumer[] = [
  new OrderCreatedConsumer(),
];

async function main() {
  await producer.connect();
  await Promise.all(consumers.map(c => c.start()));
  logger.info('All Kafka consumers started');

  process.on('SIGTERM', async () => {
    logger.info('SIGTERM received, shutting down consumers');
    await Promise.all(consumers.map(c => c.stop()));
    await producer.disconnect();
    process.exit(0);
  });
}

main().catch(console.error);
```

---

## まとめ

Claude CodeでKafkaストリーム処理を設計する：

1. **CLAUDE.md** にautoCommit false・リトライ3回→DLQ・冪等コンシューマー・グレースフルシャットダウンを明記
2. **手動オフセットコミット** で処理成功後のみオフセットを進め、再起動時の重複処理を防止
3. **Redisで冪等性管理** `processed:{eventId}` を24時間TTLで保持、同じイベントを複数コンシューマーが処理しない
4. **DLQトピック** に失敗理由・元トピック・エラー詳細をヘッダーに付けて送信、後続の再処理を可能に

---

*Kafka設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
