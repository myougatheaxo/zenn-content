---
title: "Claude CodeでRedis Streamsを設計する：イベントソーシング・コンシューマーグループ・ACK管理"
emoji: "🌊"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "eventdriven"]
published: true
published_at: "2026-03-14 07:00"
---

## はじめに

Pub/SubはメッセージをBufferして再読みできない——Redis Streamsはイベントを永続化しコンシューマーグループで複数ワーカーに振り分け、未処理メッセージの再配信もできる。Claude Codeに設計させる。

---

## CLAUDE.mdにRedis Streams設計ルールを書く

```markdown
## Redis Streams設計ルール

### ストリーム設計
- ストリーム名: {domain}.{eventType}（例: order.created）
- MAXLEN ~10000: 最新10,000件に自動トリミング（省メモリ）
- メッセージID: 自動生成（*）でタイムスタンプ埋め込み

### コンシューマーグループ
- グループ名: {service}（例: inventory-service）
- 複数ワーカーで並列処理（workerCount = CPUコア数）
- ACK: 処理完了後にXACKを送信（PELから削除）

### 未処理メッセージ再配信
- XPENDINGで30秒以上処理中のメッセージを検出
- XCLAIMで別ワーカーに再割り当て
- 3回以上失敗: DLQストリームへ移動
```

---

## Redis Streams実装の生成

```
Redis Streamsによるイベントストリーム処理を設計してください。

要件：
- ストリームへのイベント追加
- コンシューマーグループ
- ACK管理
- 未処理メッセージの自動再配信

生成ファイル: src/streams/
```

---

## 生成されるRedis Streams実装

```typescript
// src/streams/producer.ts — ストリームプロデューサー

export async function publishToStream<T extends Record<string, string>>(
  streamKey: string,
  event: T & { eventType: string; occurredAt?: string }
): Promise<string> {
  const fields = {
    ...event,
    occurredAt: event.occurredAt ?? new Date().toISOString(),
    eventId: crypto.randomUUID(),
  };

  // フィールドをRedisのkeyvalue配列に変換
  const args = Object.entries(fields).flat();

  const messageId = await redis.xadd(
    streamKey,
    'MAXLEN', '~', '10000', // 近似トリミング（パフォーマンス優先）
    '*',                      // IDは自動生成
    ...args
  );

  logger.debug({ streamKey, messageId, eventType: event.eventType }, 'Event published to stream');
  return messageId!;
}

// 使用例
await publishToStream('order.created', {
  eventType: 'order.created',
  orderId: order.id,
  userId: order.userId,
  totalAmount: String(order.totalAmount),
});
```

```typescript
// src/streams/consumer.ts — コンシューマーグループ基底クラス

interface StreamMessage {
  id: string;
  fields: Record<string, string>;
}

export abstract class StreamConsumer {
  private isRunning = false;
  private readonly deliveryCount = new Map<string, number>(); // メッセージID → 配信回数

  constructor(
    protected readonly streamKey: string,
    protected readonly groupName: string,
    protected readonly consumerName: string, // ワーカーID（ユニーク）
    protected readonly options: {
      batchSize?: number;
      blockMs?: number;
      pendingTimeoutMs?: number;
      maxDeliveries?: number;
    } = {}
  ) {}

  abstract processMessage(message: StreamMessage): Promise<void>;

  async start(): Promise<void> {
    // コンシューマーグループを作成（既存なら無視）
    try {
      await redis.xgroup('CREATE', this.streamKey, this.groupName, '$', 'MKSTREAM');
    } catch (e) {
      if (!(e as Error).message.includes('BUSYGROUP')) throw e;
    }

    this.isRunning = true;
    const batchSize = this.options.batchSize ?? 10;
    const blockMs = this.options.blockMs ?? 5000;

    while (this.isRunning) {
      // 1. まずPENDINGメッセージの再処理（前回未ACKのもの）
      await this.processPendingMessages();

      // 2. 新しいメッセージを読み取り（最大blockMsまでブロック）
      const messages = await redis.xreadgroup(
        'GROUP', this.groupName, this.consumerName,
        'COUNT', batchSize,
        'BLOCK', blockMs,
        'STREAMS', this.streamKey,
        '>' // 新しいメッセージのみ
      );

      if (!messages) continue; // タイムアウト（新着なし）

      for (const [, streamMessages] of messages) {
        for (const [id, rawFields] of streamMessages) {
          const fields = Object.fromEntries(
            Array.from({ length: rawFields.length / 2 }, (_, i) => [rawFields[i * 2], rawFields[i * 2 + 1]])
          );
          await this.handleMessage({ id, fields });
        }
      }
    }
  }

  private async handleMessage(message: StreamMessage): Promise<void> {
    const maxDeliveries = this.options.maxDeliveries ?? 3;

    try {
      await this.processMessage(message);
      // 成功: ACK送信してPELから削除
      await redis.xack(this.streamKey, this.groupName, message.id);
      this.deliveryCount.delete(message.id);
    } catch (err) {
      const count = (this.deliveryCount.get(message.id) ?? 0) + 1;
      this.deliveryCount.set(message.id, count);

      logger.error({ messageId: message.id, attempt: count, err }, 'Message processing failed');

      if (count >= maxDeliveries) {
        // 最大試行回数超過: DLQへ移動
        await this.moveToDLQ(message, count, String(err));
        await redis.xack(this.streamKey, this.groupName, message.id);
        this.deliveryCount.delete(message.id);
      }
      // ACKしない → PELに残る → 次のpendingチェックで再処理
    }
  }

  // 30秒以上処理中（PEL）のメッセージを再配信
  private async processPendingMessages(): Promise<void> {
    const pendingTimeoutMs = this.options.pendingTimeoutMs ?? 30_000;
    const pending = await redis.xpending(
      this.streamKey, this.groupName,
      'IDLE', pendingTimeoutMs,
      '-', '+', 10
    );

    for (const [messageId, consumerName] of (pending as string[][])) {
      if (consumerName === this.consumerName) continue; // 自分のはスキップ

      // 別ワーカーから自分にXCLAIM
      const claimed = await redis.xclaim(
        this.streamKey, this.groupName, this.consumerName,
        pendingTimeoutMs, messageId
      );

      for (const [id, rawFields] of claimed as [string, string[]][]) {
        const fields = Object.fromEntries(
          Array.from({ length: rawFields.length / 2 }, (_, i) => [rawFields[i * 2], rawFields[i * 2 + 1]])
        );
        logger.warn({ messageId: id, from: consumerName }, 'Reclaimed pending message');
        await this.handleMessage({ id, fields });
      }
    }
  }

  private async moveToDLQ(message: StreamMessage, deliveries: number, error: string): Promise<void> {
    await publishToStream(`${this.streamKey}.dlq`, {
      ...message.fields,
      eventType: 'dlq',
      dlqSourceStream: this.streamKey,
      dlqGroup: this.groupName,
      dlqError: error,
      dlqDeliveries: String(deliveries),
    });
    logger.error({ messageId: message.id, deliveries }, 'Message moved to DLQ');
  }

  stop(): void {
    this.isRunning = false;
  }
}
```

```typescript
// src/streams/consumers/orderConsumer.ts — 注文イベントコンシューマー

export class OrderCreatedConsumer extends StreamConsumer {
  constructor(workerId: string) {
    super('order.created', 'inventory-service', `worker-${workerId}`, {
      batchSize: 10,
      pendingTimeoutMs: 30_000,
      maxDeliveries: 3,
    });
  }

  async processMessage(message: StreamMessage): Promise<void> {
    const { orderId, userId, totalAmount } = message.fields;

    await prisma.$transaction(async (tx) => {
      await tx.inventory.update({
        where: { orderId },
        data: { status: 'RESERVED' },
      });
    });

    logger.info({ orderId, userId }, 'Inventory reserved for order');
  }
}

// 複数ワーカーを起動
const workers = Array.from({ length: 4 }, (_, i) => new OrderCreatedConsumer(String(i)));
workers.forEach(w => w.start());
```

---

## まとめ

Claude CodeでRedis Streamsを設計する：

1. **CLAUDE.md** にMAXLEN ~10000・コンシューマーグループ・ACK管理・30秒PEL再配信・DLQ移動を明記
2. **PELの自動再配信** でワーカーがクラッシュしても30秒後に他ワーカーがXCLAIMで引き継ぐ
3. **3回失敗でDLQ** — DLQストリームにエラー情報付きで転送、後続の手動修正が可能
4. **XREADGROUP BLOCK** でポーリングを避けつつ、blockMs経過後にPENDINGチェックのためのループを継続

---

*イベント設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
