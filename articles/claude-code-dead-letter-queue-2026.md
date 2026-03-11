---
title: "Claude CodeでDead Letter Queueを設計する：失敗メッセージの隔離・再処理・アラート"
emoji: "☠️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 11:00"
---

## はじめに

「メッセージ処理が3回失敗してキューが詰まった」——Dead Letter Queue（DLQ）で失敗メッセージを隔離し、手動再処理・自動診断・アラートを整備する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにDLQ設計ルールを書く

```markdown
## Dead Letter Queue設計ルール

### 失敗判定
- 最大リトライ数を超えたメッセージをDLQへ移動
- リトライ間隔: 指数バックオフ（1分, 5分, 15分, 1時間）
- 毒メッセージ（poison pill）: パース不能はリトライなしで即DLQ

### DLQの内容
- 元のメッセージ + エラー詳細 + 失敗回数 + 最終失敗時刻
- DLQメッセージはTTL 30日（監査ログとして保持）
- DLQ投入時にSlack/PagerDutyアラート

### 再処理
- 管理者がDLQメッセージを個別または一括で元のキューに戻せる
- 再処理前にメッセージを編集可能（スキーマ修正後のマイグレーション）
- 再処理にはoperatorIdと理由を記録
```

---

## DLQ実装の生成

```
Dead Letter Queueシステムを設計してください。

要件：
- 失敗メッセージの自動DLQ移動
- エラー詳細の記録
- 管理者による再処理API
- アラート連携

生成ファイル: src/messaging/dlq/
```

---

## 生成されるDLQ実装

```typescript
// src/messaging/dlq/deadLetterQueue.ts — DLQシステム

export interface DeadLetteredMessage<T = unknown> {
  dlqId: string;
  originalMessageId: string;
  queueName: string;
  payload: T;
  failureCount: number;
  lastError: string;
  lastFailedAt: Date;
  firstFailedAt: Date;
  history: Array<{ attemptedAt: Date; error: string }>;
  deadLetteredAt: Date;
}

export class DeadLetterQueue {
  private readonly dlqStreamKey = 'dlq:messages';
  private readonly dlqHashKey = 'dlq:index';

  async moveToDLQ<T>(
    message: { messageId: string; queueName: string; payload: T },
    error: Error,
    failureHistory: Array<{ attemptedAt: Date; error: string }>
  ): Promise<string> {
    const dlqId = ulid();
    const dlqMessage: DeadLetteredMessage<T> = {
      dlqId,
      originalMessageId: message.messageId,
      queueName: message.queueName,
      payload: message.payload,
      failureCount: failureHistory.length,
      lastError: error.message,
      lastFailedAt: new Date(),
      firstFailedAt: failureHistory[0]?.attemptedAt ?? new Date(),
      history: failureHistory,
      deadLetteredAt: new Date(),
    };

    // Redis Streamに追加
    await redis.xAdd(this.dlqStreamKey, '*', {
      data: JSON.stringify(dlqMessage),
    });

    // 検索インデックス
    await redis.hSet(this.dlqHashKey, dlqId, JSON.stringify(dlqMessage));

    // アラート
    await this.sendAlert(dlqMessage);

    logger.error(
      { dlqId, messageId: message.messageId, queueName: message.queueName, error: error.message },
      'Message moved to DLQ'
    );

    return dlqId;
  }

  private async sendAlert<T>(dlqMessage: DeadLetteredMessage<T>): Promise<void> {
    await slackClient.postMessage({
      channel: '#alerts-dlq',
      text: 'DLQにメッセージが追加されました',
      blocks: [
        {
          type: 'section',
          fields: [
            { type: 'mrkdwn', text: `*Queue:* ${dlqMessage.queueName}` },
            { type: 'mrkdwn', text: `*失敗回数:* ${dlqMessage.failureCount}` },
            { type: 'mrkdwn', text: `*最終エラー:* ${dlqMessage.lastError.slice(0, 100)}` },
            { type: 'mrkdwn', text: `*DLQ ID:* ${dlqMessage.dlqId}` },
          ],
        },
      ],
    });
  }

  async list(options: { queueName?: string; limit?: number; offset?: number } = {}): Promise<{
    messages: DeadLetteredMessage[];
    total: number;
  }> {
    const all = await redis.hGetAll(this.dlqHashKey);
    let messages = Object.values(all).map(v => JSON.parse(v) as DeadLetteredMessage);

    if (options.queueName) {
      messages = messages.filter(m => m.queueName === options.queueName);
    }

    messages.sort((a, b) => new Date(b.deadLetteredAt).getTime() - new Date(a.deadLetteredAt).getTime());

    const total = messages.length;
    const limit = options.limit ?? 50;
    const offset = options.offset ?? 0;

    return { messages: messages.slice(offset, offset + limit), total };
  }

  async reprocess(dlqId: string, operatorId: string, reason: string, payloadOverride?: unknown): Promise<void> {
    const raw = await redis.hGet(this.dlqHashKey, dlqId);
    if (!raw) throw new NotFoundError(`DLQ message ${dlqId} not found`);

    const dlqMessage = JSON.parse(raw) as DeadLetteredMessage;
    const payload = payloadOverride ?? dlqMessage.payload;

    // 元のキューに戻す
    await redis.xAdd(`queue:${dlqMessage.queueName}`, '*', {
      messageId: ulid(),
      payload: JSON.stringify(payload),
      reprocessedFrom: dlqId,
      reprocessedBy: operatorId,
    });

    // DLQから削除
    await redis.hDel(this.dlqHashKey, dlqId);

    // 監査ログ
    await prisma.dlqReprocessLog.create({
      data: { dlqId, operatorId, reason, originalQueueName: dlqMessage.queueName, reprocessedAt: new Date() },
    });

    logger.info({ dlqId, operatorId, reason }, 'DLQ message reprocessed');
  }
}
```

```typescript
// src/messaging/dlq/retryableConsumer.ts — リトライ付きコンシューマー

export interface RetryConfig {
  maxAttempts: number;
  backoffMs: number[];
}

export class RetryableConsumer<T> {
  private readonly dlq = new DeadLetterQueue();

  constructor(
    private readonly queueName: string,
    private readonly config: RetryConfig = {
      maxAttempts: 4,
      backoffMs: [60_000, 300_000, 900_000, 3_600_000],
    }
  ) {}

  async consume(
    message: { messageId: string; payload: T },
    handler: (payload: T) => Promise<void>
  ): Promise<void> {
    const historyKey = `retry:history:${message.messageId}`;
    const historyRaw = await redis.get(historyKey);
    const history: Array<{ attemptedAt: Date; error: string }> = historyRaw
      ? JSON.parse(historyRaw)
      : [];

    try {
      await handler(message.payload);
      await redis.del(historyKey);
    } catch (error) {
      const errorMessage = (error as Error).message;
      history.push({ attemptedAt: new Date(), error: errorMessage });

      if (history.length >= this.config.maxAttempts) {
        // 最大リトライ超過: DLQへ
        await this.dlq.moveToDLQ(
          { messageId: message.messageId, queueName: this.queueName, payload: message.payload },
          error as Error,
          history
        );
        await redis.del(historyKey);
      } else {
        // 次のリトライをスケジュール
        const nextDelayMs = this.config.backoffMs[history.length - 1] ?? 3_600_000;
        await redis.set(historyKey, JSON.stringify(history), { PX: nextDelayMs * 2 });

        // 遅延キューに追加（Sorted Set: score = 実行予定時刻）
        await redis.zAdd(`queue:${this.queueName}:delayed`, {
          score: Date.now() + nextDelayMs,
          value: JSON.stringify({ messageId: message.messageId, payload: message.payload }),
        });

        logger.warn(
          { messageId: message.messageId, attempt: history.length, nextRetryMs: nextDelayMs },
          'Message will be retried'
        );
      }
    }
  }
}

// 管理者API
router.get('/api/admin/dlq', requireAdmin, async (req, res) => {
  const dlq = new DeadLetterQueue();
  const result = await dlq.list({ queueName: req.query.queue as string });
  res.json(result);
});

router.post('/api/admin/dlq/:dlqId/reprocess', requireAdmin, async (req, res) => {
  const dlq = new DeadLetterQueue();
  await dlq.reprocess(req.params.dlqId, req.user.id, req.body.reason, req.body.payloadOverride);
  res.json({ message: 'Requeued successfully' });
});
```

---

## まとめ

Claude CodeでDead Letter Queueを設計する：

1. **CLAUDE.md** に最大リトライ数超過でDLQ移動・指数バックオフ（1分/5分/15分/1時間）・DLQ投入時にSlackアラート・30日TTLを明記
2. **失敗履歴の記録** で`history`配列にattemptedAt+errorを蓄積——DLQ調査時に「何回目の試行でどんなエラーが出たか」を全て確認できる
3. **管理者再処理API** で`payloadOverride`を許容——スキーマバグ修正後に「修正済みペイロードで再処理」が可能。operatorIdと理由を監査ログに残す
4. **遅延キュー（Sorted Set）** でバックオフを実装——`score = Date.now() + nextDelayMs`でZRANGEBYSCOREが時刻到達したメッセージだけを取り出す

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
