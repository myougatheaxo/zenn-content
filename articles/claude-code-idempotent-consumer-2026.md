---
title: "Claude Codeで冪等コンシューマーを設計する：重複メッセージ処理・処理済みID管理・at-least-once保証"
emoji: "🔁"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 10:00"
---

## はじめに

「メッセージブローカーの再配信で同じ注文が2回処理された」——at-least-once配信を前提とした冪等コンシューマーで、重複メッセージを安全に処理する設計をClaude Codeに生成させる。

---

## CLAUDE.mdに冪等コンシューマー設計ルールを書く

```markdown
## 冪等コンシューマー設計ルール

### メッセージID追跡
- 全メッセージにグローバルユニークID（ULID/UUID）を付与
- 処理済みIDをRedisに保存（TTL: 7日 = メッセージ再配信窓口の2倍）
- 処理前にRedis SET NX でロック取得（二重処理防止）

### 処理結果キャッシュ
- 成功した処理の結果をキャッシュ（同じIDが再来たら結果を返す）
- 失敗した処理は未処理扱い（リトライを許可）
- キャッシュミスかつロック取得できない場合は待機

### データベース側の冪等性
- INSERT ... ON CONFLICT DO NOTHING（重複挿入を無視）
- UPDATE ... WHERE processed_message_id != 'id'（条件付き更新）
```

---

## 冪等コンシューマー実装の生成

```
冪等コンシューマーを設計してください。

要件：
- Redis SET NX による重複排除
- 処理結果キャッシュ
- DB側の冪等INSERT
- デッドレターキュー連携

生成ファイル: src/messaging/idempotent/
```

---

## 生成される冪等コンシューマー実装

```typescript
// src/messaging/idempotent/idempotentConsumer.ts — 冪等コンシューマー基盤

export interface Message<T = unknown> {
  messageId: string;      // グローバルユニーク
  payload: T;
  publishedAt: string;
}

export class IdempotentConsumer<T> {
  private readonly lockTtlMs = 30_000;  // 処理ロック30秒
  private readonly deduplicateTtlSec = 7 * 86400; // 処理済み7日保持

  async process(
    message: Message<T>,
    handler: (payload: T) => Promise<unknown>
  ): Promise<{ status: 'processed' | 'duplicate' | 'lock-failed' }> {
    const lockKey = `idempotent:lock:${message.messageId}`;
    const doneKey = `idempotent:done:${message.messageId}`;
    const resultKey = `idempotent:result:${message.messageId}`;

    // 1. 処理済みチェック
    const alreadyDone = await redis.exists(doneKey);
    if (alreadyDone) {
      logger.debug({ messageId: message.messageId }, 'Duplicate message — skipping');
      return { status: 'duplicate' };
    }

    // 2. 処理ロック取得（SET NX PX）
    const lockAcquired = await redis.set(lockKey, '1', { NX: true, PX: this.lockTtlMs });
    if (!lockAcquired) {
      // 他のワーカーが処理中
      logger.warn({ messageId: message.messageId }, 'Lock not acquired — another worker processing');
      return { status: 'lock-failed' };
    }

    try {
      // 3. ハンドラー実行
      const result = await handler(message.payload);

      // 4. 処理済みマーク（結果もキャッシュ）
      const pipeline = redis.pipeline();
      pipeline.set(doneKey, '1', { EX: this.deduplicateTtlSec });
      if (result !== undefined) {
        pipeline.set(resultKey, JSON.stringify(result), { EX: this.deduplicateTtlSec });
      }
      await pipeline.exec();

      logger.info({ messageId: message.messageId }, 'Message processed successfully');
      return { status: 'processed' };
    } catch (error) {
      // 失敗時はロック解放（リトライを許可）
      logger.error({ messageId: message.messageId, error }, 'Message processing failed');
      throw error;
    } finally {
      // ロック解放
      await redis.del(lockKey);
    }
  }
}
```

```typescript
// src/messaging/idempotent/orderEventConsumer.ts — 注文イベントコンシューマー

export class OrderEventConsumer {
  private readonly consumer = new IdempotentConsumer<OrderCreatedPayload>();

  async handleOrderCreated(message: Message<OrderCreatedPayload>): Promise<void> {
    const result = await this.consumer.process(message, async (payload) => {
      // DB側も冪等：ON CONFLICT DO NOTHING
      await prisma.$executeRaw`
        INSERT INTO order_confirmations (order_id, confirmed_at, message_id)
        VALUES (${payload.orderId}, NOW(), ${message.messageId})
        ON CONFLICT (order_id) DO NOTHING
      `;

      // 在庫予約（冪等キー付き）
      await inventoryService.reserve({
        idempotencyKey: message.messageId,  // 同じキーで再呼び出し→ノーオプ
        orderId: payload.orderId,
        items: payload.items,
      });

      // メール送信（冪等キー付き）
      await emailService.sendOrderConfirmation({
        idempotencyKey: message.messageId,
        userId: payload.userId,
        orderId: payload.orderId,
      });

      return { orderId: payload.orderId, status: 'confirmed' };
    });

    if (result.status === 'duplicate') {
      // 重複は正常（再配信を無視）
      metrics.duplicateMessages.inc({ consumer: 'order-created' });
    }

    if (result.status === 'lock-failed') {
      // ロック取得失敗：少し待ってNACK（ブローカーに再配信させる）
      await sleep(2000);
      throw new LockConflictError('Will retry — another worker processing');
    }
  }
}
```

```typescript
// src/messaging/idempotent/idempotentRepository.ts — DB側冪等INSERT

export class IdempotentRepository {
  // 処理済みメッセージIDをDBに永続化（Redisだけでは不安な場合）
  async markProcessed(messageId: string, entityId: string, result: unknown): Promise<boolean> {
    try {
      await prisma.processedMessage.create({
        data: {
          messageId,
          entityId,
          result: JSON.stringify(result),
          processedAt: new Date(),
        },
      });
      return true; // 新規処理
    } catch (error: any) {
      if (error.code === 'P2002') {
        // ユニーク制約違反 = 重複
        return false;
      }
      throw error;
    }
  }

  // DBトランザクション内で冪等性を保証
  async processWithinTransaction<T>(
    messageId: string,
    fn: (tx: PrismaClient) => Promise<T>
  ): Promise<{ result: T; isDuplicate: boolean }> {
    return prisma.$transaction(async (tx) => {
      // 処理済みチェック（FOR UPDATE でロック）
      const existing = await tx.$queryRaw<Array<{ result: string }>>`
        SELECT result FROM processed_messages WHERE message_id = ${messageId} FOR UPDATE
      `.then(rows => rows[0]);

      if (existing) {
        return { result: JSON.parse(existing.result) as T, isDuplicate: true };
      }

      const result = await fn(tx);

      await tx.processedMessage.create({
        data: { messageId, result: JSON.stringify(result), processedAt: new Date() },
      });

      return { result, isDuplicate: false };
    });
  }
}
```

---

## まとめ

Claude Codeで冪等コンシューマーを設計する：

1. **CLAUDE.md** に処理済みIDをRedis SET NXでロック・7日TTLで保持・失敗時はロック解放してリトライ許可を明記
2. **Redis SET NX** で「先に取得したワーカーだけが処理を実行」——ロック取得失敗は正常（他ワーカーが処理中）。少し待ってNACKで再配信
3. **DB側の冪等性** `INSERT ... ON CONFLICT DO NOTHING` と `processedMessages`テーブルで二重挿入を防止——Redisが落ちてもDBが担保
4. **下流APIへの冪等キー伝播** でinventory/emailなど外部サービスへの呼び出しにもmessageIdを渡す——メッセージの再配信が連鎖した場合でも外部副作用が重複しない

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
