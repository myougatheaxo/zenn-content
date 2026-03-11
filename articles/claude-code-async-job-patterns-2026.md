---
title: "Claude Codeで非同期ジョブパターンを設計する：長時間処理の非同期化・ポーリング・Webhook通知"
emoji: "⏳"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-19 12:00"
---

## はじめに

「PDF生成処理が30秒かかってタイムアウトする」「レポート生成中にユーザーが待ちきれず何度もボタンを押す」——長時間処理を非同期ジョブとして切り出し、ポーリングまたはWebhookで完了を通知する設計をClaude Codeに生成させる。

---

## CLAUDE.mdに非同期ジョブパターン設計ルールを書く

```markdown
## 非同期ジョブパターン設計ルール

### ジョブ登録
- 長時間処理（>2秒）は非同期ジョブに変換
- 即座にジョブIDを返す（202 Accepted）
- ジョブIDでステータスをポーリング可能

### ステータス管理
- PENDING → RUNNING → COMPLETED / FAILED
- 実行中は進捗率（0-100%）も提供
- 失敗時はエラー詳細を記録

### 通知
- ポーリング: GET /jobs/{id} でステータス確認
- Webhook: 完了時にクライアントのURLにPOST
- SSE: クライアントが接続中はリアルタイムでプッシュ
```

---

## 非同期ジョブ実装の生成

```
非同期ジョブシステムを設計してください。

要件：
- ジョブキューとワーカー
- ステータスポーリングAPI
- Webhook通知
- 進捗追跡

生成ファイル: src/jobs/
```

---

## 生成される非同期ジョブ実装

```typescript
// src/jobs/jobManager.ts — ジョブ管理

export type JobStatus = 'pending' | 'running' | 'completed' | 'failed';

export interface Job<TInput = unknown, TOutput = unknown> {
  id: string;
  type: string;
  status: JobStatus;
  input: TInput;
  output?: TOutput;
  progress: number;       // 0-100
  errorMessage?: string;
  createdAt: Date;
  startedAt?: Date;
  completedAt?: Date;
  webhookUrl?: string;    // 完了時の通知先
}

export class JobManager {
  // ジョブを登録（即座にIDを返す）
  async enqueue<TInput>(jobType: string, input: TInput, options: {
    webhookUrl?: string;
    priority?: 'normal' | 'high';
  } = {}): Promise<{ jobId: string }> {
    const jobId = ulid();
    const job: Job<TInput> = {
      id: jobId,
      type: jobType,
      status: 'pending',
      input,
      progress: 0,
      createdAt: new Date(),
      webhookUrl: options.webhookUrl,
    };

    // DBに保存
    await prisma.job.create({
      data: {
        id: jobId,
        type: jobType,
        status: 'pending',
        input: JSON.stringify(input),
        progress: 0,
        createdAt: new Date(),
        webhookUrl: options.webhookUrl,
      },
    });

    // ジョブキューに追加
    const queueKey = options.priority === 'high' ? 'queue:jobs:high' : 'queue:jobs:normal';
    await redis.xAdd(queueKey, '*', {
      jobId,
      jobType,
      payload: JSON.stringify(input),
    });

    logger.info({ jobId, jobType }, 'Job enqueued');
    return { jobId };
  }

  // ジョブステータスを取得
  async getStatus(jobId: string): Promise<Job | null> {
    const row = await prisma.job.findUnique({ where: { id: jobId } });
    if (!row) return null;

    return {
      id: row.id,
      type: row.type,
      status: row.status as JobStatus,
      input: JSON.parse(row.input),
      output: row.output ? JSON.parse(row.output) : undefined,
      progress: row.progress,
      errorMessage: row.errorMessage ?? undefined,
      createdAt: row.createdAt,
      startedAt: row.startedAt ?? undefined,
      completedAt: row.completedAt ?? undefined,
      webhookUrl: row.webhookUrl ?? undefined,
    };
  }

  // 進捗を更新（ワーカーから呼び出す）
  async updateProgress(jobId: string, progress: number): Promise<void> {
    await prisma.job.update({
      where: { id: jobId },
      data: { progress: Math.min(100, Math.max(0, progress)) },
    });

    // SSEクライアントに通知
    sseManager.broadcast(jobId, { type: 'progress', progress });
  }

  // ジョブ完了（ワーカーから呼び出す）
  async complete<TOutput>(jobId: string, output: TOutput): Promise<void> {
    const job = await prisma.job.update({
      where: { id: jobId },
      data: {
        status: 'completed',
        output: JSON.stringify(output),
        progress: 100,
        completedAt: new Date(),
      },
    });

    // SSEクライアントに通知
    sseManager.broadcast(jobId, { type: 'completed', output });

    // Webhook通知
    if (job.webhookUrl) {
      await this.sendWebhook(job.webhookUrl, {
        jobId,
        status: 'completed',
        output,
        completedAt: new Date().toISOString(),
      });
    }
  }

  async fail(jobId: string, errorMessage: string): Promise<void> {
    const job = await prisma.job.update({
      where: { id: jobId },
      data: { status: 'failed', errorMessage, completedAt: new Date() },
    });

    sseManager.broadcast(jobId, { type: 'failed', errorMessage });

    if (job.webhookUrl) {
      await this.sendWebhook(job.webhookUrl, { jobId, status: 'failed', errorMessage });
    }
  }

  private async sendWebhook(url: string, payload: unknown): Promise<void> {
    try {
      await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'X-Myouga-Event': 'job.completed' },
        body: JSON.stringify(payload),
        signal: AbortSignal.timeout(5000),
      });
    } catch (error) {
      logger.error({ url, error }, 'Webhook delivery failed');
    }
  }
}

// APIエンドポイント
const jobManager = new JobManager();

// ジョブ登録: 202 Acceptedで即座にIDを返す
router.post('/api/reports/generate', requireAuth, async (req, res) => {
  const { startDate, endDate, webhookUrl } = req.body;

  const { jobId } = await jobManager.enqueue('generate-report', {
    startDate, endDate, userId: req.user.id,
  }, { webhookUrl });

  res.status(202).json({
    jobId,
    statusUrl: `/api/jobs/${jobId}`,
    message: 'Report generation started',
  });
});

// ポーリング: ステータス確認
router.get('/api/jobs/:jobId', requireAuth, async (req, res) => {
  const job = await jobManager.getStatus(req.params.jobId);

  if (!job) return res.status(404).json({ error: 'Job not found' });

  res.json({
    jobId: job.id,
    status: job.status,
    progress: job.progress,
    output: job.status === 'completed' ? job.output : undefined,
    error: job.status === 'failed' ? job.errorMessage : undefined,
    createdAt: job.createdAt,
    completedAt: job.completedAt,
  });
});

// SSE: リアルタイム進捗ストリーム
router.get('/api/jobs/:jobId/stream', requireAuth, async (req, res) => {
  res.set({
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });

  sseManager.subscribe(req.params.jobId, (event) => {
    res.write(`data: ${JSON.stringify(event)}\n\n`);
    if (event.type === 'completed' || event.type === 'failed') {
      res.end();
    }
  });

  req.on('close', () => sseManager.unsubscribe(req.params.jobId));
});
```

---

## まとめ

Claude Codeで非同期ジョブパターンを設計する：

1. **CLAUDE.md** に2秒超の処理は非同期ジョブ化・202 Acceptedでジョブを即座に返す・ポーリング/Webhook/SSEの3方式で完了を通知を明記
2. **202 Accepted + jobId** パターンでクライアントを即座に解放——PDF生成中に「30秒待ってください」ではなく「ジョブを受け付けました。`/api/jobs/01H...`で確認してください」とすぐ返答
3. **ポーリング→SSE→Webhook** の3方式で異なるクライアントに対応——ブラウザはSSEでリアルタイム更新、バックエンドサービスはWebhookで自動通知、古いクライアントはポーリング
4. **進捗率（0-100%）** でUXを改善——「処理中...」ではなく「45%完了」を表示することでユーザーの不安を軽減。updateProgress()を処理の節目に呼ぶ

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
