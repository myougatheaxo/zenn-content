---
title: "Claude Codeでcronジョブを設計する：スケジューラー・重複防止・監視"
emoji: "⏰"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "cron", "bullmq"]
published: true
---

## はじめに

cronジョブで「毎分実行されるのに前の処理がまだ終わっていない」「複数サーバーで二重実行される」——この問題をClaude Codeに解決させる。

---

## CLAUDE.mdにcronルールを書く

```markdown
## cronジョブ設計ルール

### 重複防止（必須）
- 同一ジョブの二重実行を防ぐ（分散ロック）
- Redisのnx+exでロック実装（TTLはジョブ最大実行時間の2倍）
- 前のジョブが終わっていなければスキップ（エラーにしない）

### エラー処理
- 失敗時はSlack/PagerDuty通知
- 連続失敗N回でアラート昇格
- ジョブ実行履歴をDBに記録

### スケジューラー
- node-cronまたはBullMQ repeatableジョブを使う
- cronスケジュールはenv変数で設定可能にする
- タイムゾーンを明示的に指定（デフォルトUTCを避ける）
```

---

## cronジョブの生成

```
定期実行ジョブを設計してください。

ジョブ一覧:
- 毎日AM3時: 期限切れセッションの削除
- 毎時0分: サブスクリプション有効期限チェック
- 毎分: ペンディング中の通知を送信

要件：
- 分散ロック（複数サーバーで二重実行しない）
- 実行履歴をDB記録
- 失敗時にSlack通知

生成ファイル: src/jobs/
```

---

## 生成されるcronジョブ基盤

```typescript
// src/jobs/jobRunner.ts
import cron from 'node-cron';
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });

interface JobConfig {
  name: string;
  schedule: string;
  handler: () => Promise<void>;
  timeoutMs: number; // ロックTTL = timeoutMs * 2
}

export async function runWithLock(
  jobName: string,
  timeoutMs: number,
  handler: () => Promise<void>
): Promise<void> {
  const lockKey = `cron:lock:${jobName}`;
  const lockTtl = Math.ceil((timeoutMs * 2) / 1000); // 秒単位

  // 分散ロック取得
  const acquired = await redis.set(lockKey, '1', {
    NX: true,    // 存在しない場合のみ
    EX: lockTtl, // TTL
  });

  if (!acquired) {
    logger.info({ jobName }, 'Job already running, skipping');
    return;
  }

  const startedAt = new Date();
  try {
    await handler();
    await recordJobRun(jobName, 'success', startedAt);
  } catch (err) {
    await recordJobRun(jobName, 'failed', startedAt, String(err));
    await notifySlack(`❌ Cron job failed: ${jobName}\n${err}`);
    throw err;
  } finally {
    await redis.del(lockKey);
  }
}

async function recordJobRun(
  jobName: string,
  status: 'success' | 'failed',
  startedAt: Date,
  error?: string
): Promise<void> {
  await prisma.cronJobRun.create({
    data: {
      jobName,
      status,
      startedAt,
      finishedAt: new Date(),
      error,
    },
  });
}
```

```typescript
// src/jobs/definitions.ts
import cron from 'node-cron';
import { runWithLock } from './jobRunner';
import { cleanupExpiredSessions } from './handlers/cleanupSessions';
import { checkSubscriptionExpiry } from './handlers/subscriptionCheck';
import { sendPendingNotifications } from './handlers/notifications';

const TIMEZONE = process.env.CRON_TIMEZONE ?? 'Asia/Tokyo';

export function startCronJobs(): void {
  // 毎日AM3時: 期限切れセッション削除
  cron.schedule(
    process.env.CLEANUP_SCHEDULE ?? '0 3 * * *',
    () => runWithLock('cleanup-sessions', 30 * 60 * 1000, cleanupExpiredSessions),
    { timezone: TIMEZONE }
  );

  // 毎時: サブスクリプション有効期限チェック
  cron.schedule(
    '0 * * * *',
    () => runWithLock('subscription-check', 10 * 60 * 1000, checkSubscriptionExpiry),
    { timezone: TIMEZONE }
  );

  // 毎分: 通知送信
  cron.schedule(
    '* * * * *',
    () => runWithLock('send-notifications', 50 * 1000, sendPendingNotifications),
    { timezone: TIMEZONE }
  );

  logger.info('Cron jobs started');
}
```

---

## BullMQ Repeatableジョブ版

```typescript
// src/jobs/bullmqCron.ts
import { Queue, Worker } from 'bullmq';

const cronQueue = new Queue('cron-jobs', { connection: redisConnection });

// Repeatableジョブの登録（サーバー起動時に1回だけ）
export async function registerRepeatableJobs(): Promise<void> {
  await cronQueue.add(
    'cleanup-sessions',
    {},
    {
      repeat: { pattern: '0 3 * * *', tz: 'Asia/Tokyo' },
      jobId: 'cleanup-sessions', // 重複登録防止
    }
  );

  await cronQueue.add(
    'subscription-check',
    {},
    {
      repeat: { pattern: '0 * * * *', tz: 'Asia/Tokyo' },
      jobId: 'subscription-check',
    }
  );
}

// Worker（このプロセスが実際の処理を担当）
const worker = new Worker(
  'cron-jobs',
  async (job) => {
    switch (job.name) {
      case 'cleanup-sessions':
        return cleanupExpiredSessions();
      case 'subscription-check':
        return checkSubscriptionExpiry();
      default:
        logger.warn({ jobName: job.name }, 'Unknown cron job');
    }
  },
  { connection: redisConnection, concurrency: 1 } // 同時実行1（直列化）
);
```

---

## まとめ

Claude CodeでcronジョブをCLAUDE.mdで設計する：

1. **CLAUDE.md** に重複防止・エラー処理・タイムゾーン指定を明記
2. **Redisロック（NX+EX）** で複数サーバーでの二重実行を防ぐ
3. **実行履歴DB記録** で障害調査を容易にする
4. **BullMQ repeatable** でスケジューラーの管理をシンプルにする

---

*cronジョブの問題（重複実行リスク、ロック未実装）を検出するスキルは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
