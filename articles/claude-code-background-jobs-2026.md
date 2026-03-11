---
title: "Claude CodeでBullMQジョブキューを設計する：バックグラウンド処理の標準パターン"
emoji: "⚙️"
type: "tech"
topics: ["claudecode", "bullmq", "redis", "nodejs", "バックグラウンド処理"]
published: true
---

## はじめに

メール送信・画像処理・外部APIコール——これらをリクエストスレッドで実行するとタイムアウトのリスクがある。BullMQでジョブキューを構築し、Claude Codeに設計を任せる。

---

## CLAUDE.mdにジョブキュールールを書く

```markdown
## ジョブキュールール

### 使用場面（必須）
- レスポンスに3秒以上かかる処理はキューに移す
- 外部API呼び出し（メール送信・決済・Webhook）
- ファイル処理（画像リサイズ・PDF生成）
- バッチ処理（レポート生成・データエクスポート）

### ライブラリ
- BullMQ（Redis Streams使用、型安全）
- Redis 7以上

### 設計ルール
- ジョブの型定義は必須（JobData interfaceを定義）
- リトライ: 最大3回、指数バックオフ
- タイムアウト: ジョブごとに設定（default: 30秒）
- ジョブ完了後はデータを保持しない（removeOnComplete: true）
- 失敗ジョブは100件まで保持（デバッグ用）

### キュー命名規則
- {domain}-{action}: `email-send`, `image-resize`, `report-generate`

### モニタリング
- Bull Boardで管理画面を提供
- 失敗率が5%を超えたらアラート
```

---

## BullMQワーカーの生成

```
BullMQを使ったジョブキューシステムを生成してください。

要件：
- ジョブ: メール送信（email-send）
- ジョブデータ型: { to: string, subject: string, body: string, userId: string }
- リトライ: 最大3回、指数バックオフ（1000ms * 2^retryCount）
- タイムアウト: 30秒
- 失敗時にログを記録
- TypeScript

生成するファイル:
- src/queues/emailQueue.ts（キュー定義）
- src/workers/emailWorker.ts（ワーカー）
- src/queues/types.ts（ジョブデータ型）
```

---

## 生成されるコードの例

```typescript
// src/queues/types.ts
export interface EmailJobData {
  to: string;
  subject: string;
  body: string;
  userId: string;
}

export interface ImageResizeJobData {
  inputPath: string;
  outputPath: string;
  width: number;
  height: number;
}
```

```typescript
// src/queues/emailQueue.ts
import { Queue } from 'bullmq';
import { EmailJobData } from './types';
import { redis } from '../config/redis';

export const emailQueue = new Queue<EmailJobData>('email-send', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: true,
    removeOnFail: 100,
  },
});

export async function enqueueEmail(data: EmailJobData) {
  return emailQueue.add('send', data, { timeout: 30_000 });
}
```

```typescript
// src/workers/emailWorker.ts
import { Worker, Job } from 'bullmq';
import { EmailJobData } from '../queues/types';
import { sendEmail } from '../services/emailService';
import { logger } from '../lib/logger';

export const emailWorker = new Worker<EmailJobData>(
  'email-send',
  async (job: Job<EmailJobData>) => {
    const { to, subject, body, userId } = job.data;

    logger.info({ userId, to, jobId: job.id }, 'Sending email');

    await sendEmail({ to, subject, html: body });

    logger.info({ userId, to, jobId: job.id }, 'Email sent successfully');
  },
  {
    connection: redis,
    concurrency: 5,
  }
);

emailWorker.on('failed', (job, err) => {
  logger.error(
    { jobId: job?.id, userId: job?.data.userId, error: err.message },
    'Email job failed'
  );
});
```

---

## 進捗状況の追跡

```
ジョブの進捗状況をAPIで返す仕組みを生成してください。

要件：
- POST /jobs/export でジョブ作成（jobIdを返す）
- GET /jobs/:jobId/status でステータス確認
  レスポンス: { status: 'waiting' | 'active' | 'completed' | 'failed', progress: number }
- ジョブ内でjob.updateProgress(0-100)を呼ぶ
- 完了後は結果のダウンロードURLも返す

実装先: src/routes/jobs.ts
```

```typescript
// 生成されるAPIの例
router.post('/export', async (req, res) => {
  const job = await reportQueue.add('generate', {
    userId: req.user.id,
    filters: req.body.filters,
  });
  res.json({ jobId: job.id });
});

router.get('/:jobId/status', async (req, res) => {
  const job = await reportQueue.getJob(req.params.jobId);
  if (!job) return res.status(404).json({ error: 'Job not found' });

  const state = await job.getState();
  const progress = job.progress;

  res.json({
    status: state,
    progress,
    ...(state === 'completed' && { downloadUrl: `/downloads/${job.returnvalue}` }),
    ...(state === 'failed' && { error: job.failedReason }),
  });
});
```

---

## Bull Boardで管理画面を追加

```
Bull BoardのWebUIをExpressに追加してください。

要件：
- パス: /admin/queues
- Basic認証で保護（ADMIN_USER/ADMIN_PASSの環境変数）
- 監視するキュー: email-send, image-resize, report-generate
```

---

## まとめ

Claude CodeでBullMQジョブキューを設計する：

1. **CLAUDE.md** に使用場面・命名規則・リトライポリシーを明記
2. **ジョブ型定義** を先に生成させる（TypeScriptの型安全）
3. **キュー + ワーカー** をペアで生成
4. **進捗API** でフロントからジョブ状態を追跡
5. **Bull Board** で運用時の可視性を確保

---

*バックグラウンドジョブの設計レビューは **Code Review Pack（¥980）** の `/code-review` で自動化できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
