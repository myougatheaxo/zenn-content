---
title: "Claude Codeで競合コンシューマーを設計する：水平スケール・排他処理・ワーカープール管理"
emoji: "🏃"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 18:00"
---

## はじめに

「処理が増えてきたのでWorkerを増やしたが、同じメッセージが複数のWorkerで二重処理された」——競合コンシューマーパターンで複数のWorkerが安全にタスクを奪い合い、水平スケールを実現する設計をClaude Codeに生成させる。

---

## CLAUDE.mdに競合コンシューマー設計ルールを書く

```markdown
## 競合コンシューマー設計ルール

### 排他処理
- Redis Consumer GroupでWorker間の排他的メッセージ割り当て
- Consumer Groupは各メッセージを1つのWorkerにのみ配信
- 処理完了後にACK（未ACKメッセージはタイムアウト後に再割り当て）

### ワーカー管理
- ワーカー数はキュー深度に応じて自動スケール
- ヘルスチェック: 30秒ごとに生存確認
- 落ちたワーカーのPELを他のワーカーが引き継ぐ

### グレースフルシャットダウン
- SIGTERMを受信したらキュー取得を停止
- 処理中のタスクが完了するまで待機（最大30秒）
- タイムアウト後は強制終了
```

---

## 競合コンシューマー実装の生成

```
競合コンシューマーパターンを設計してください。

要件：
- Redis Consumer Group
- Pending Entry List（PEL）の引き継ぎ
- グレースフルシャットダウン
- 自動スケーリング

生成ファイル: src/messaging/competingConsumers/
```

---

## 生成される競合コンシューマー実装

```typescript
// src/messaging/competingConsumers/consumerGroup.ts — Consumer Group Worker

export interface ConsumerGroupConfig {
  streamKey: string;      // Redis Streamキー
  groupName: string;      // Consumer Group名
  consumerName: string;   // このWorkerの識別子
  batchSize: number;      // 1回に取得するメッセージ数
  blockMs: number;        // ブロッキング読み取りのタイムアウト
  pendingTimeoutMs: number; // PELタイムアウト（この時間未ACKは再割り当て）
}

export class ConsumerGroupWorker<T> {
  private running = false;
  private activeProcessing = 0;
  private readonly maxGracefulWaitMs = 30_000;

  constructor(
    private readonly config: ConsumerGroupConfig,
    private readonly handler: (payload: T) => Promise<void>
  ) {}

  async start(): Promise<void> {
    // Consumer Groupがなければ作成
    try {
      await redis.xGroupCreate(
        this.config.streamKey,
        this.config.groupName,
        '$',  // 新規メッセージからのみ処理（既存メッセージは無視）
        { MKSTREAM: true }
      );
    } catch (error: any) {
      if (!error.message.includes('BUSYGROUP')) throw error;
      // 既に存在する場合は続行
    }

    this.running = true;
    logger.info({ consumer: this.config.consumerName, group: this.config.groupName }, 'Consumer started');

    // グレースフルシャットダウンのシグナルハンドラー
    process.on('SIGTERM', () => this.stop());
    process.on('SIGINT', () => this.stop());

    // まず未処理のPELを引き継ぐ
    await this.claimPendingMessages();

    // メインループ
    while (this.running) {
      await this.processNextBatch();
    }

    // グレースフルシャットダウン: 処理中のタスクを待つ
    await this.waitForActiveProcessing();
  }

  private async processNextBatch(): Promise<void> {
    const messages = await redis.xReadGroup(
      this.config.groupName,
      this.config.consumerName,
      { key: this.config.streamKey, id: '>' },
      { COUNT: this.config.batchSize, BLOCK: this.config.blockMs }
    );

    if (!messages || messages[0].messages.length === 0) return;

    const processingPromises = messages[0].messages.map(msg =>
      this.processMessage(msg)
    );

    await Promise.allSettled(processingPromises);
  }

  private async processMessage(msg: { id: string; message: Record<string, string> }): Promise<void> {
    this.activeProcessing++;

    try {
      const payload = JSON.parse(msg.message.payload) as T;

      await this.handler(payload);

      // 処理成功: ACK
      await redis.xAck(this.config.streamKey, this.config.groupName, msg.id);

      metrics.messagesProcessed.inc({ consumer: this.config.consumerName });
      logger.debug({ messageId: msg.id, consumer: this.config.consumerName }, 'Message acknowledged');
    } catch (error) {
      logger.error({ messageId: msg.id, consumer: this.config.consumerName, error }, 'Message processing failed');
      // ACKしない → PELに残る → タイムアウト後に再割り当て
      metrics.messagesFailed.inc({ consumer: this.config.consumerName });
    } finally {
      this.activeProcessing--;
    }
  }

  // 落ちたConsumerのPEL（未ACKメッセージ）を引き継ぐ
  private async claimPendingMessages(): Promise<void> {
    let startId = '0';

    while (true) {
      // タイムアウト超過の未ACKメッセージを自分のものにする
      const claimed = await redis.xAutoClaim(
        this.config.streamKey,
        this.config.groupName,
        this.config.consumerName,
        this.config.pendingTimeoutMs,
        startId,
        { COUNT: 100 }
      );

      if (claimed.messages.length === 0) break;

      logger.info({ count: claimed.messages.length }, 'Claimed pending messages from dead consumers');

      for (const msg of claimed.messages) {
        await this.processMessage(msg);
      }

      startId = claimed.nextId;
      if (startId === '0') break;
    }
  }

  async stop(): Promise<void> {
    logger.info({ consumer: this.config.consumerName }, 'Graceful shutdown initiated');
    this.running = false;
  }

  private async waitForActiveProcessing(): Promise<void> {
    const startMs = Date.now();

    while (this.activeProcessing > 0) {
      if (Date.now() - startMs > this.maxGracefulWaitMs) {
        logger.warn({ activeProcessing: this.activeProcessing }, 'Graceful shutdown timeout — forcing exit');
        break;
      }
      await sleep(100);
    }

    logger.info({ consumer: this.config.consumerName }, 'Consumer stopped');
  }
}
```

```typescript
// src/messaging/competingConsumers/workerPool.ts — ワーカープール管理

export class WorkerPool<T> {
  private workers: Map<string, ConsumerGroupWorker<T>> = new Map();

  constructor(
    private readonly streamKey: string,
    private readonly groupName: string,
    private readonly handler: (payload: T) => Promise<void>
  ) {}

  async scale(targetWorkers: number): Promise<void> {
    const currentCount = this.workers.size;

    if (targetWorkers > currentCount) {
      // スケールアップ
      for (let i = currentCount; i < targetWorkers; i++) {
        const consumerName = `worker-${process.pid}-${i}`;
        const worker = new ConsumerGroupWorker<T>(
          {
            streamKey: this.streamKey,
            groupName: this.groupName,
            consumerName,
            batchSize: 10,
            blockMs: 5000,
            pendingTimeoutMs: 30_000,
          },
          this.handler
        );

        this.workers.set(consumerName, worker);
        worker.start().catch(err => logger.error({ consumerName, err }, 'Worker error'));
        logger.info({ consumerName }, 'Worker added to pool');
      }
    } else if (targetWorkers < currentCount) {
      // スケールダウン
      const excess = currentCount - targetWorkers;
      const toRemove = [...this.workers.keys()].slice(-excess);

      for (const name of toRemove) {
        const worker = this.workers.get(name);
        await worker?.stop();
        this.workers.delete(name);
        logger.info({ consumerName: name }, 'Worker removed from pool');
      }
    }
  }

  // キュー深度に応じた自動スケーリング
  async autoScale(options: { minWorkers: number; maxWorkers: number; targetLag: number }): Promise<void> {
    const info = await redis.xInfoGroups(this.streamKey);
    const group = info.find((g: any) => g.name === this.groupName);
    const lag = group?.lag ?? 0;

    let target = this.workers.size;

    if (lag > options.targetLag * 2 && this.workers.size < options.maxWorkers) {
      target = Math.min(this.workers.size + 2, options.maxWorkers);
    } else if (lag < options.targetLag / 2 && this.workers.size > options.minWorkers) {
      target = Math.max(this.workers.size - 1, options.minWorkers);
    }

    if (target !== this.workers.size) {
      logger.info({ from: this.workers.size, to: target, lag }, 'Auto-scaling worker pool');
      await this.scale(target);
    }
  }
}

// 使用例
const pool = new WorkerPool<OrderPayload>(
  'queue:order-processing',
  'order-processors',
  async (order) => {
    await processOrder(order);
  }
);

// 最初は2ワーカーで開始
await pool.scale(2);

// 30秒ごとに自動スケール（lag > 100でスケールアップ）
setInterval(() => pool.autoScale({ minWorkers: 1, maxWorkers: 10, targetLag: 100 }), 30_000);
```

---

## まとめ

Claude Codeで競合コンシューマーを設計する：

1. **CLAUDE.md** にRedis Consumer GroupでWorker間排他割り当て・ACK後にPELから削除・タイムアウト超過で他Workerが引き継ぎ・SIGTERMで処理中タスク完了まで待機を明記
2. **`xAutoClaim`（PEL引き継ぎ）** で落ちたWorkerの未ACKメッセージを自動引き継ぎ——Workerがクラッシュしても`pendingTimeoutMs`後に他のWorkerが処理を再開
3. **グレースフルシャットダウン** でSIGTERM受信後に新規取得を停止し、処理中タスクが完了するまで最大30秒待機——Kubernetes Podの入れ替え時にメッセージロストなし
4. **自動スケーリング（`autoScale`）** でキューのlag（遅延）に応じてWorker数を動的調整——lag > targetLag×2でスケールアップ、lag < targetLag/2でスケールダウン

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
