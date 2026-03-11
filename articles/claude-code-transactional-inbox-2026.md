---
title: "Claude CodeでTransactional Inboxを設計する：メッセージ受信保証・重複排除・at-least-once処理"
emoji: "📥"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 12:00"
---

## はじめに

「外部WebhookをDBに保存する前にアプリがクラッシュして、イベントを取りこぼした」——Transactional Inboxパターンでメッセージの受信をDBトランザクションと同期し、at-least-once処理を保証する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにTransactional Inbox設計ルールを書く

```markdown
## Transactional Inbox設計ルール

### 受信フロー
1. メッセージを受信したら即座にinboxテーブルへINSERT（DBトランザクション内）
2. 受信ACKをブローカーに返す（データは確実にDBにある）
3. 別のワーカーがinboxを定期的にポーリングして処理

### 重複排除
- message_idにUNIQUE制約（同じメッセージIDが来ても1回だけ処理）
- 処理済みはstatus='processed'に更新（削除しない）
- 30日後にバッチで物理削除

### 再処理
- 処理中にクラッシュした場合: status='pending'のまま残る
- 再起動後にワーカーが再処理（冪等処理が前提）
- 5回失敗したらstatus='failed'に更新してアラート
```

---

## Transactional Inbox実装の生成

```
Transactional Inboxを設計してください。

要件：
- メッセージのDB永続化
- 重複排除
- at-least-once処理保証
- ポーリングワーカー

生成ファイル: src/messaging/inbox/
```

---

## 生成されるTransactional Inbox実装

```typescript
// src/messaging/inbox/inboxRepository.ts — Inboxリポジトリ

export interface InboxMessage {
  id: string;
  messageId: string;      // 外部メッセージID（UNIQUE）
  source: string;         // 送信元（例: 'stripe', 'github-webhook'）
  eventType: string;
  payload: unknown;
  status: 'pending' | 'processing' | 'processed' | 'failed';
  receivedAt: Date;
  processedAt?: Date;
  attemptCount: number;
  lastError?: string;
}

export class InboxRepository {
  // メッセージをinboxに保存（重複は無視）
  async receive(message: {
    messageId: string;
    source: string;
    eventType: string;
    payload: unknown;
  }): Promise<{ inserted: boolean; inboxId: string }> {
    try {
      const result = await prisma.inboxMessage.create({
        data: {
          messageId: message.messageId,
          source: message.source,
          eventType: message.eventType,
          payload: JSON.stringify(message.payload),
          status: 'pending',
          receivedAt: new Date(),
          attemptCount: 0,
        },
      });

      return { inserted: true, inboxId: result.id };
    } catch (error: any) {
      if (error.code === 'P2002') {
        // UNIQUE制約違反 = 重複メッセージ（冪等性）
        const existing = await prisma.inboxMessage.findUnique({
          where: { messageId: message.messageId },
          select: { id: true },
        });
        return { inserted: false, inboxId: existing!.id };
      }
      throw error;
    }
  }

  // 処理待ちメッセージをバッチで取得（FOR UPDATE SKIP LOCKED）
  async claimBatch(options: { limit: number; maxAttempts: number }): Promise<InboxMessage[]> {
    return prisma.$queryRaw<InboxMessage[]>`
      UPDATE inbox_messages
      SET status = 'processing', attempt_count = attempt_count + 1
      WHERE id IN (
        SELECT id FROM inbox_messages
        WHERE status = 'pending'
          AND attempt_count < ${options.maxAttempts}
        ORDER BY received_at ASC
        LIMIT ${options.limit}
        FOR UPDATE SKIP LOCKED
      )
      RETURNING *
    `;
  }

  async markProcessed(id: string): Promise<void> {
    await prisma.inboxMessage.update({
      where: { id },
      data: { status: 'processed', processedAt: new Date() },
    });
  }

  async markFailed(id: string, error: Error): Promise<void> {
    await prisma.inboxMessage.update({
      where: { id },
      data: { status: 'failed', lastError: error.message },
    });
  }

  async returnToPending(id: string): Promise<void> {
    await prisma.inboxMessage.update({
      where: { id },
      data: { status: 'pending' },
    });
  }
}
```

```typescript
// src/messaging/inbox/inboxWorker.ts — Inboxポーリングワーカー

type InboxHandler = (message: InboxMessage) => Promise<void>;

export class InboxWorker {
  private readonly repository = new InboxRepository();
  private readonly handlers = new Map<string, InboxHandler>();
  private running = false;

  register(eventType: string, handler: InboxHandler): void {
    this.handlers.set(eventType, handler);
  }

  async start(): Promise<void> {
    this.running = true;
    logger.info('Inbox worker started');

    while (this.running) {
      const processed = await this.processBatch();

      if (processed === 0) {
        // 処理するものがない場合: 少し待つ
        await sleep(5000);
      }
    }
  }

  private async processBatch(): Promise<number> {
    const messages = await this.repository.claimBatch({ limit: 10, maxAttempts: 5 });

    if (messages.length === 0) return 0;

    await Promise.allSettled(
      messages.map(msg => this.processOne(msg))
    );

    return messages.length;
  }

  private async processOne(message: InboxMessage): Promise<void> {
    const handler = this.handlers.get(message.eventType);

    if (!handler) {
      logger.warn({ eventType: message.eventType, messageId: message.messageId }, 'No handler registered — skipping');
      await this.repository.markProcessed(message.id);
      return;
    }

    try {
      await handler(message);
      await this.repository.markProcessed(message.id);

      logger.info(
        { messageId: message.messageId, eventType: message.eventType, attempt: message.attemptCount },
        'Inbox message processed'
      );
    } catch (error) {
      if (message.attemptCount >= 5) {
        await this.repository.markFailed(message.id, error as Error);
        logger.error({ messageId: message.messageId, error }, 'Inbox message permanently failed');
        // Slackアラート
      } else {
        await this.repository.returnToPending(message.id);
        logger.warn({ messageId: message.messageId, attempt: message.attemptCount, error }, 'Inbox message failed — will retry');
      }
    }
  }
}

// Webhook受信エンドポイント
router.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  const sig = req.headers['stripe-signature'] as string;
  const event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);

  const inbox = new InboxRepository();

  // DBトランザクション内でinboxに保存（クラッシュ耐性）
  const { inserted } = await inbox.receive({
    messageId: event.id,         // StripeのイベントID（UUID）
    source: 'stripe',
    eventType: event.type,
    payload: event.data,
  });

  if (!inserted) {
    logger.debug({ eventId: event.id }, 'Duplicate Stripe webhook — ignored');
  }

  // 即座に200を返してStripeのリトライを防ぐ
  res.status(200).json({ received: true });
});

// ワーカー起動
const worker = new InboxWorker();
worker.register('payment_intent.succeeded', handlePaymentSucceeded);
worker.register('payment_intent.payment_failed', handlePaymentFailed);
worker.start();
```

---

## まとめ

Claude CodeでTransactional Inboxを設計する：

1. **CLAUDE.md** に受信即座にinboxへINSERT→ブローカーにACK・ワーカーが定期ポーリングで処理・重複はUNIQUE制約で1回処理・5回失敗でalertを明記
2. **受信とACKの分離** でWebhookを受け取ったら即座にDBに保存してから200を返す——「保存する前にクラッシュ」問題を解消。StripeのリトライがあっługてもUNIQUE制約が重複を防ぐ
3. **FOR UPDATE SKIP LOCKED** でワーカー並列実行時の重複処理を防止——複数Workerが同じバッチを取得しても、PostgreSQLのロックで排他的に1WorkerだけがNEXTを取得
4. **`returnToPending`とリトライカウント** で過渡的な障害を自動回復——ハンドラーが一時的に失敗しても`pending`に戻して再試行。5回失敗したら`failed`に遷移して手動調査

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
