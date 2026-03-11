---
title: "Claude Codeで優先度付きキューを設計する：タスク優先度・SLA保証・スターベーション防止"
emoji: "⬆️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 16:00"
---

## はじめに

「有料ユーザーのジョブが無料ユーザーのジョブに埋もれて遅延している」——優先度付きキューで重要なタスクを先に処理し、低優先度タスクもスターベーション（無期限待機）しない設計をClaude Codeに生成させる。

---

## CLAUDE.mdに優先度付きキュー設計ルールを書く

```markdown
## 優先度付きキュー設計ルール

### 優先度レベル
- CRITICAL (0): インシデント対応・緊急決済処理 → 5分以内
- HIGH (1): 有料ユーザーのジョブ → 30分以内
- NORMAL (2): 無料ユーザーのジョブ → 2時間以内
- LOW (3): バッチ処理・レポート生成 → 24時間以内

### スターベーション防止
- 待機時間が閾値を超えたタスクは自動的に優先度を上げる
- LOW優先度が24時間以上待機したらNORMALに昇格
- ワーカーは高優先度キューを先にチェック（ポーリング順で制御）

### SLA追跡
- タスクの優先度とエンキュー時刻からSLA期限を計算
- SLA違反リスクが高いタスクをアラート
```

---

## 優先度付きキュー実装の生成

```
優先度付きキューシステムを設計してください。

要件：
- 優先度レベル（CRITICAL/HIGH/NORMAL/LOW）
- スターベーション防止
- SLA追跡とアラート
- 公平スケジューリング

生成ファイル: src/queue/priority/
```

---

## 生成される優先度付きキュー実装

```typescript
// src/queue/priority/priorityQueue.ts — 優先度付きキュー

export type Priority = 'CRITICAL' | 'HIGH' | 'NORMAL' | 'LOW';

const PRIORITY_ORDER: Priority[] = ['CRITICAL', 'HIGH', 'NORMAL', 'LOW'];

const SLA_MS: Record<Priority, number> = {
  CRITICAL: 5 * 60_000,       // 5分
  HIGH: 30 * 60_000,          // 30分
  NORMAL: 2 * 60 * 60_000,    // 2時間
  LOW: 24 * 60 * 60_000,      // 24時間
};

const STARVATION_THRESHOLD_MS: Record<Priority, number> = {
  CRITICAL: Infinity,         // 昇格なし
  HIGH: Infinity,             // 昇格なし
  NORMAL: 4 * 60 * 60_000,   // 4時間でHIGHに昇格
  LOW: 24 * 60 * 60_000,     // 24時間でNORMALに昇格
};

export interface QueueTask<T = unknown> {
  taskId: string;
  priority: Priority;
  payload: T;
  enqueuedAt: Date;
  slaDeadline: Date;
  userId?: string;
  tenantId?: string;
}

export class PriorityQueue {
  private getQueueKey(priority: Priority): string {
    return `pqueue:${priority.toLowerCase()}`;
  }

  async enqueue<T>(task: {
    taskId: string;
    priority: Priority;
    payload: T;
    userId?: string;
    tenantId?: string;
  }): Promise<void> {
    const enqueuedAt = new Date();
    const slaDeadline = new Date(enqueuedAt.getTime() + SLA_MS[task.priority]);

    const queueTask: QueueTask<T> = {
      taskId: task.taskId,
      priority: task.priority,
      payload: task.payload,
      enqueuedAt,
      slaDeadline,
      userId: task.userId,
      tenantId: task.tenantId,
    };

    // Sorted Set: score = エンキュー時刻（同優先度内ではFIFO）
    await redis.zAdd(this.getQueueKey(task.priority), {
      score: enqueuedAt.getTime(),
      value: JSON.stringify(queueTask),
    });

    // タスクデータをHashに保存（メタデータ参照用）
    await redis.hSet(`pqueue:task:${task.taskId}`, {
      priority: task.priority,
      enqueuedAt: enqueuedAt.toISOString(),
      slaDeadline: slaDeadline.toISOString(),
    });

    logger.debug({ taskId: task.taskId, priority: task.priority, slaDeadline }, 'Task enqueued');
    metrics.queueDepth.inc({ priority: task.priority });
  }

  // 高優先度から順にデキュー
  async dequeue<T>(): Promise<QueueTask<T> | null> {
    // スターベーション防止: 先に昇格チェック
    await this.promoteStarvedTasks();

    for (const priority of PRIORITY_ORDER) {
      const queueKey = this.getQueueKey(priority);

      // 最も古いタスクを取得（score最小）
      const items = await redis.zRangeWithScores(queueKey, 0, 0);
      if (items.length === 0) continue;

      const taskData = items[0].value;
      await redis.zRem(queueKey, taskData);

      const task = JSON.parse(taskData) as QueueTask<T>;

      // SLA違反チェック
      if (new Date() > task.slaDeadline) {
        metrics.slaViolations.inc({ priority: task.priority });
        logger.warn({ taskId: task.taskId, priority: task.priority }, 'SLA violation — task dequeued after deadline');
      }

      metrics.queueDepth.dec({ priority: task.priority });
      return task;
    }

    return null; // 全キュー空
  }

  // スターベーション防止: 長時間待機タスクを昇格
  private async promoteStarvedTasks(): Promise<void> {
    const now = Date.now();

    // NORMAL → HIGH昇格
    await this.promoteIfStarved('NORMAL', 'HIGH', now - STARVATION_THRESHOLD_MS.NORMAL);
    // LOW → NORMAL昇格
    await this.promoteIfStarved('LOW', 'NORMAL', now - STARVATION_THRESHOLD_MS.LOW);
  }

  private async promoteIfStarved(from: Priority, to: Priority, cutoffScore: number): Promise<void> {
    // cutoffScore以前にエンキューされたタスクを取得
    const starvedItems = await redis.zRangeByScore(this.getQueueKey(from), 0, cutoffScore);

    if (starvedItems.length === 0) return;

    const pipeline = redis.pipeline();
    for (const item of starvedItems) {
      const task = JSON.parse(item) as QueueTask;
      const promoted = { ...task, priority: to };

      pipeline.zRem(this.getQueueKey(from), item);
      pipeline.zAdd(this.getQueueKey(to), {
        score: task.enqueuedAt.getTime(),
        value: JSON.stringify(promoted),
      });
    }
    await pipeline.exec();

    logger.info({ count: starvedItems.length, from, to }, 'Starved tasks promoted');
    metrics.starvationPromotions.inc({ from, to, count: starvedItems.length });
  }

  // SLAリスク監視（定期実行）
  async checkSlaRisk(): Promise<void> {
    const warnBeforeMs = 5 * 60_000; // SLA期限5分前にアラート
    const now = Date.now();

    for (const priority of PRIORITY_ORDER) {
      const items = await redis.zRangeWithScores(this.getQueueKey(priority), 0, -1);

      for (const { value } of items) {
        const task = JSON.parse(value) as QueueTask;
        const timeToSla = new Date(task.slaDeadline).getTime() - now;

        if (timeToSla > 0 && timeToSla < warnBeforeMs) {
          logger.warn({ taskId: task.taskId, priority, timeToSlaMs: timeToSla }, 'SLA at risk');
          metrics.slaAtRisk.inc({ priority });
        }
      }
    }
  }
}

// ワーカー実装
export class PriorityWorker {
  private readonly queue = new PriorityQueue();
  private running = false;

  async start(): Promise<void> {
    this.running = true;
    while (this.running) {
      const task = await this.queue.dequeue();

      if (!task) {
        await sleep(1000); // キュー空の場合は1秒待機
        continue;
      }

      try {
        await this.process(task);
      } catch (error) {
        logger.error({ taskId: task.taskId, error }, 'Task processing failed');
      }
    }
  }

  private async process(task: QueueTask): Promise<void> {
    logger.info({ taskId: task.taskId, priority: task.priority }, 'Processing task');
    // 実際の処理...
  }
}
```

---

## まとめ

Claude Codeで優先度付きキューを設計する：

1. **CLAUDE.md** にCRITICAL/HIGH/NORMAL/LOWの4優先度・SLA時間（5分/30分/2時間/24時間）・スターベーション昇格閾値を明記
2. **Sorted Set（score = エンキュー時刻）** で同優先度内ではFIFO順を維持——優先度キューをSorted Setで実装することで`zRangeByScore`で一定時間以上待機したタスクを効率的に取得できる
3. **スターベーション防止** `promoteIfStarved()`でデキュー前に自動昇格——LOW優先度タスクが24時間以上待機するとNORMALに昇格、さらに4時間待機でHIGHに昇格
4. **SLAリスク監視** でSLA期限5分前にアラート——デキューワーカーと別に定期Cronでチェックし「処理されずに期限が迫っているタスク」を早期検出

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
