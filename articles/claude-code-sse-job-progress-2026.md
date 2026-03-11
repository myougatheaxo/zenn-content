---
title: "Claude CodeでSSEによるジョブ進捗ストリーミングを設計する：リアルタイム進捗・BullMQ連携"
emoji: "📡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "realtime"]
published: true
published_at: "2026-03-15 15:00"
---

## はじめに

「レポート生成中...」でぐるぐるするUIを終わらせる——Server-Sent Events（SSE）でバックグラウンドジョブの進捗をリアルタイムにクライアントへ配信する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにSSEジョブ進捗設計ルールを書く

```markdown
## SSEジョブ進捗ストリーミング設計ルール

### SSE設計
- エンドポイント: GET /api/jobs/{jobId}/progress
- Content-Type: text/event-stream
- 接続タイムアウト: 5分（長時間ジョブ用）
- Heartbeat: 15秒ごとにcommentイベント（プロキシのタイムアウト対策）

### 進捗伝達
- BullMQジョブからRedis Pub/Subへ進捗をpublish
- SSEエンドポイントがsubscribeして転送
- イベント種別: progress（0-100）, log（ログメッセージ）, complete, error

### クリーンアップ
- クライアント切断時はsubscriptionを即解除
- ジョブ完了後15秒でSSE接続を自動クローズ
```

---

## SSEジョブ進捗の生成

```
SSEによるバックグラウンドジョブ進捗ストリーミングを設計してください。

要件：
- SSEエンドポイント
- BullMQからRedisへ進捗Publish
- Heartbeat
- 接続クリーンアップ

生成ファイル: src/jobs/progress/
```

---

## 生成されるSSEジョブ進捗実装

```typescript
// src/jobs/progress/publisher.ts — ジョブからの進捗送信

export class JobProgressPublisher {
  private readonly channelPrefix = 'job:progress';

  constructor(private readonly jobId: string) {}

  // 進捗パーセント更新（0-100）
  async reportProgress(percent: number, message?: string): Promise<void> {
    await redis.publish(`${this.channelPrefix}:${this.jobId}`, JSON.stringify({
      type: 'progress',
      percent: Math.min(100, Math.max(0, percent)),
      message,
      timestamp: Date.now(),
    }));
  }

  // ログメッセージ送信
  async reportLog(level: 'info' | 'warn' | 'error', message: string): Promise<void> {
    await redis.publish(`${this.channelPrefix}:${this.jobId}`, JSON.stringify({
      type: 'log',
      level,
      message,
      timestamp: Date.now(),
    }));
  }

  // 完了通知
  async reportComplete(result?: unknown): Promise<void> {
    await redis.publish(`${this.channelPrefix}:${this.jobId}`, JSON.stringify({
      type: 'complete',
      result,
      timestamp: Date.now(),
    }));
  }

  // エラー通知
  async reportError(error: string): Promise<void> {
    await redis.publish(`${this.channelPrefix}:${this.jobId}`, JSON.stringify({
      type: 'error',
      error,
      timestamp: Date.now(),
    }));
  }
}

// BullMQワーカーでの使用例
export async function reportGenerationWorker(job: Job) {
  const progress = new JobProgressPublisher(job.id!);

  try {
    await progress.reportLog('info', 'レポート生成開始');
    await progress.reportProgress(10, 'データ取得中...');

    const data = await fetchReportData(job.data.reportId);
    await progress.reportProgress(40, 'データ集計中...');

    const aggregated = await aggregateData(data);
    await progress.reportProgress(70, 'PDFレンダリング中...');

    const pdfBuffer = await generatePDF(aggregated);
    await progress.reportProgress(90, 'S3にアップロード中...');

    const url = await uploadToS3(pdfBuffer, `reports/${job.id}.pdf`);
    await progress.reportProgress(100, '完了');
    await progress.reportComplete({ url });
  } catch (err) {
    await progress.reportError((err as Error).message);
    throw err; // BullMQのリトライのために再スロー
  }
}
```

```typescript
// src/jobs/progress/sseEndpoint.ts — SSEエンドポイント

import { Redis } from 'ioredis';

// Redisサブスクリプション用の別接続（subscribe中は他のコマンドを実行できないため）
function createSubscriberConnection(): Redis {
  return new Redis(process.env.REDIS_URL!);
}

router.get('/api/jobs/:jobId/progress', requireAuth, async (req, res) => {
  const { jobId } = req.params;

  // ジョブの所有権確認
  const job = await prisma.job.findFirst({
    where: { id: jobId, userId: req.user.id },
  });
  if (!job) return res.sendStatus(404);

  // 既に完了しているジョブはRedisから最終状態を返す
  const finalState = await redis.get(`job:final:${jobId}`);
  if (finalState) {
    const state = JSON.parse(finalState);
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'X-Accel-Buffering': 'no', // Nginx バッファリング無効化
    });
    res.write(`data: ${finalState}\n\n`);
    return res.end();
  }

  // SSEレスポンス開始
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no',
  });

  const subscriber = createSubscriberConnection();
  const channel = `job:progress:${jobId}`;

  // Heartbeat（15秒ごと）— プロキシのタイムアウト対策
  const heartbeat = setInterval(() => {
    if (!res.closed) res.write(': heartbeat\n\n');
  }, 15_000);

  // 5分後に自動切断
  const autoClose = setTimeout(() => {
    res.write(`data: ${JSON.stringify({ type: 'timeout', message: '接続タイムアウト' })}\n\n`);
    cleanup();
  }, 5 * 60 * 1000);

  let isCleanedUp = false;

  const cleanup = async () => {
    if (isCleanedUp) return;
    isCleanedUp = true;
    clearInterval(heartbeat);
    clearTimeout(autoClose);
    await subscriber.unsubscribe(channel);
    subscriber.disconnect();
    if (!res.closed) res.end();
  };

  subscriber.subscribe(channel);

  subscriber.on('message', async (_, message) => {
    if (res.closed) { await cleanup(); return; }

    res.write(`data: ${message}\n\n`);

    // 完了/エラーイベントは15秒後に切断
    const event = JSON.parse(message);
    if (event.type === 'complete' || event.type === 'error') {
      // 最終状態をRedisに保存（再接続時に返すため）
      await redis.set(`job:final:${jobId}`, message, { EX: 3600 }); // 1時間保持
      setTimeout(cleanup, 15_000);
    }
  });

  // クライアント切断時のクリーンアップ
  req.on('close', cleanup);
});
```

```typescript
// フロントエンド（React）での使用例

function JobProgressModal({ jobId }: { jobId: string }) {
  const [progress, setProgress] = useState(0);
  const [logs, setLogs] = useState<string[]>([]);
  const [status, setStatus] = useState<'running' | 'complete' | 'error'>('running');
  const [resultUrl, setResultUrl] = useState<string | null>(null);

  useEffect(() => {
    const es = new EventSource(`/api/jobs/${jobId}/progress`, { withCredentials: true });

    es.onmessage = (event) => {
      const data = JSON.parse(event.data);

      switch (data.type) {
        case 'progress':
          setProgress(data.percent);
          if (data.message) setLogs(prev => [...prev, data.message]);
          break;
        case 'log':
          setLogs(prev => [...prev, `[${data.level}] ${data.message}`]);
          break;
        case 'complete':
          setStatus('complete');
          setResultUrl(data.result?.url);
          es.close();
          break;
        case 'error':
          setStatus('error');
          setLogs(prev => [...prev, `エラー: ${data.error}`]);
          es.close();
          break;
      }
    };

    return () => es.close(); // コンポーネントアンマウント時に切断
  }, [jobId]);

  return (
    <div>
      <progress value={progress} max={100} />
      <p>{progress}%</p>
      {logs.map((log, i) => <p key={i}>{log}</p>)}
      {status === 'complete' && <a href={resultUrl!}>ダウンロード</a>}
    </div>
  );
}
```

---

## まとめ

Claude CodeでSSEジョブ進捗ストリーミングを設計する：

1. **CLAUDE.md** にSSE専用Redisサブスクリプション接続・15秒Heartbeat・5分自動切断・最終状態Redis保存を明記
2. **Redis Pub/Sub** でジョブプロセスとSSE接続を疎結合——複数サーバーにまたがっても機能する
3. **完了状態のRedis保存** で遅れて接続したクライアントも最終状態を受け取れる（再接続対応）
4. **X-Accel-Buffering: no** でNginxのバッファリングを無効化——SSEがリアルタイムで届く

---

*リアルタイム設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
