---
title: "Claude CodeでMessage Queueを設計する：BullMQ・SQS・デッドレターキュー"
emoji: "📨"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "aws"]
published: true
---

## はじめに

画像処理・メール送信・外部API呼び出しをリクエスト内で同期実行するとタイムアウトする。Message Queueで非同期処理に切り出す。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにMessage Queue設計ルールを書く

```markdown
## Message Queue設計ルール

### キュー選択
- ローカル/単一サーバー: BullMQ（Redis、シンプル）
- 本番/高可用性: AWS SQS（マネージド、スケーラブル）
- 高スループット: SQS + Lambda（ポーリング不要）

### ジョブ設計
- ジョブは冪等性を持つ（リトライされる前提）
- ペイロードは最小限（IDのみ渡してDBから取得）
- タイムアウト設定（ハング防止）

### リトライ・失敗処理
- 指数バックオフでリトライ（3回: 1分→5分→30分）
- 最大リトライ後はDeadLetterQueue（DLQ）に移動
- DLQは定期監視・アラート設定
- 失敗ジョブの詳細ログ保存（デバッグ用）
```

---

## BullMQジョブキューの生成

```
BullMQで非同期ジョブキューシステムを設計してください。

ユースケース: 画像のリサイズ処理
要件：
- Redisバックエンド
- 指数バックオフリトライ（3回）
- 失敗時のデッドレターキュー
- 進行状況の追跡
- ジョブ完了通知

生成ファイル: src/queues/imageProcessor.ts
```

---

## 生成されるBullMQ実装

```typescript
// src/queues/imageProcessor.ts
import { Queue, Worker, Job, QueueEvents } from 'bullmq';
import { createClient } from 'redis';
import sharp from 'sharp';

const connection = { host: process.env.REDIS_HOST, port: 6379 };

// ジョブのペイロード型
interface ImageProcessJob {
  imageId: string;
  userId: string;
  operations: Array<
    | { type: 'resize'; width: number; height: number }
    | { type: 'convert'; format: 'webp' | 'avif' | 'jpeg' }
    | { type: 'compress'; quality: number }
  >;
}

// キュー定義
export const imageQueue = new Queue<ImageProcessJob>('image-processing', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 60_000, // 1分→5分→25分
    },
    removeOnComplete: { count: 1000 }, // 最新1000件だけ保持
    removeOnFail: false, // 失敗ジョブは残す（デバッグ用）
  },
});

// ワーカー定義
export const imageWorker = new Worker<ImageProcessJob>(
  'image-processing',
  async (job: Job<ImageProcessJob>) => {
    const { imageId, operations } = job.data;

    logger.info({ jobId: job.id, imageId }, 'Processing image');

    // DBから画像メタデータ取得
    const image = await prisma.image.findUniqueOrThrow({ where: { id: imageId } });

    // S3から元画像ダウンロード
    const originalBuffer = await downloadFromS3(image.s3Key);

    let pipeline = sharp(originalBuffer);
    let outputFormat = 'jpeg';

    for (const [index, op] of operations.entries()) {
      // 進行状況を更新（0-100%）
      await job.updateProgress(Math.round((index / operations.length) * 80));

      if (op.type === 'resize') {
        pipeline = pipeline.resize(op.width, op.height, {
          fit: 'inside',
          withoutEnlargement: true,
        });
      } else if (op.type === 'convert') {
        outputFormat = op.format;
        pipeline = pipeline.toFormat(op.format);
      } else if (op.type === 'compress') {
        pipeline = pipeline.jpeg({ quality: op.quality });
      }
    }

    const processedBuffer = await pipeline.toBuffer();

    // S3にアップロード
    const processedKey = `processed/${imageId}.${outputFormat}`;
    await uploadToS3(processedKey, processedBuffer);

    // DBを更新
    await prisma.image.update({
      where: { id: imageId },
      data: {
        processedKey,
        processedAt: new Date(),
        status: 'processed',
      },
    });

    await job.updateProgress(100);

    return { processedKey, size: processedBuffer.length };
  },
  {
    connection,
    concurrency: 5, // 同時処理数
    limiter: {
      max: 100,
      duration: 60_000, // 1分あたり100ジョブ
    },
  }
);

// エラーハンドリング
imageWorker.on('failed', async (job, err) => {
  logger.error({ jobId: job?.id, err, data: job?.data }, 'Image processing failed');

  // 最大リトライ後のDLQ処理
  if (job && job.attemptsMade >= (job.opts.attempts ?? 3)) {
    await prisma.failedJob.create({
      data: {
        queue: 'image-processing',
        jobId: job.id!,
        payload: job.data,
        error: err.message,
        failedAt: new Date(),
      },
    });

    // アラート送信（Slackなど）
    await sendAlert(`Image processing job ${job.id} failed permanently: ${err.message}`);
  }
});

imageWorker.on('completed', (job, result) => {
  logger.info({ jobId: job.id, result }, 'Image processing completed');
});
```

---

## APIからジョブを追加

```typescript
// src/routes/images.ts
router.post('/images/:id/process', authenticate, async (req, res) => {
  const image = await prisma.image.findUniqueOrThrow({
    where: { id: req.params.id, userId: req.user.id },
  });

  const job = await imageQueue.add(
    'process',
    {
      imageId: image.id,
      userId: req.user.id,
      operations: req.body.operations,
    },
    {
      jobId: `image:${image.id}`, // 重複防止（同一IDは1つだけ）
    }
  );

  res.status(202).json({
    jobId: job.id,
    status: 'queued',
    // ポーリングURL
    statusUrl: `/api/jobs/${job.id}/status`,
  });
});

// ジョブ状態確認
router.get('/jobs/:jobId/status', authenticate, async (req, res) => {
  const job = await imageQueue.getJob(req.params.jobId);
  if (!job) return res.status(404).json({ error: 'Job not found' });

  const state = await job.getState();
  const progress = job.progress;

  res.json({ jobId: job.id, state, progress });
});
```

---

## AWS SQS版（本番環境）

```typescript
// src/queues/sqsWorker.ts
import { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } from '@aws-sdk/client-sqs';

const sqs = new SQSClient({ region: 'ap-northeast-1' });
const QUEUE_URL = process.env.SQS_IMAGE_QUEUE_URL!;
const DLQ_URL = process.env.SQS_IMAGE_DLQ_URL!; // 自動でDLQに移動（SQS設定）

async function processSQSMessages(): Promise<void> {
  while (true) {
    const response = await sqs.send(
      new ReceiveMessageCommand({
        QueueUrl: QUEUE_URL,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20, // Long polling（空ポーリング削減）
        VisibilityTimeout: 300, // 5分間他のワーカーに見えない
      })
    );

    if (!response.Messages?.length) continue;

    await Promise.allSettled(
      response.Messages.map(async (message) => {
        try {
          const job = JSON.parse(message.Body!) as ImageProcessJob;
          await processImage(job);

          // 処理成功: キューから削除
          await sqs.send(
            new DeleteMessageCommand({
              QueueUrl: QUEUE_URL,
              ReceiptHandle: message.ReceiptHandle!,
            })
          );
        } catch (err) {
          logger.error({ err, messageId: message.MessageId }, 'SQS message processing failed');
          // 削除しない → Visibility Timeout後に再試行
          // MaxReceiveCount超過後はSQSがDLQに自動移動
        }
      })
    );
  }
}
```

---

## まとめ

Claude CodeでMessage Queueを設計する：

1. **CLAUDE.md** にキュー選択基準・リトライ戦略・DLQ方針を明記
2. **BullMQ** でRedisバックエンドの非同期処理（開発〜中規模本番）
3. **指数バックオフ** でリトライ（即時リトライは同じ失敗を繰り返す）
4. **AWS SQS** でマネージド・高可用性キュー（Long polling + DLQ自動移動）

---

*Message Queue設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
