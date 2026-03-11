---
title: "Claude Codeでイベントリプレイを設計する：プロジェクション再構築・バグ修正後の状態修復・スナップショット連携"
emoji: "⏪"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 14:00"
---

## はじめに

「バグで集計テーブルの値がずれた。イベントから再計算したい」——Event Sourcingのイベントログからプロジェクション（読み取りモデル）を再構築し、バグ修正後の状態を正確に復元する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにイベントリプレイ設計ルールを書く

```markdown
## イベントリプレイ設計ルール

### リプレイの安全性
- リプレイ中は読み取りモデルのbuildVersionで管理
- 古いバージョンと新しいバージョンを並行して持ち、切り替え後に古いを削除
- リプレイ中もサービスは停止しない（Blue-Green的アプローチ）

### バッチ処理
- 全イベントを一度にメモリに乗せない（ページング）
- チェックポイントを定期的に保存（障害時に途中から再開）
- スナップショットがある場合はそこからスタート

### 冪等性
- プロジェクション更新は冪等（同じイベントを2回リプレイしても同じ結果）
- INSERT ... ON CONFLICT DO UPDATEでアップサート
```

---

## イベントリプレイ実装の生成

```
イベントリプレイシステムを設計してください。

要件：
- バッチ処理（ページング）
- チェックポイント管理
- ゼロダウンタイムリプレイ
- 進捗モニタリング

生成ファイル: src/eventsourcing/replay/
```

---

## 生成されるイベントリプレイ実装

```typescript
// src/eventsourcing/replay/eventReplayer.ts — イベントリプレイエンジン

export interface ReplayOptions {
  projectionName: string;
  fromVersion?: number;     // 開始バージョン（省略時は最初から）
  toVersion?: number;       // 終了バージョン（省略時は最新まで）
  batchSize?: number;       // 1バッチのイベント数
  checkpointEvery?: number; // この件数ごとにチェックポイント保存
}

export interface ReplayProgress {
  projectionName: string;
  totalEvents: number;
  processedEvents: number;
  lastCheckpoint: number;
  startedAt: Date;
  estimatedCompletionAt?: Date;
  status: 'running' | 'completed' | 'failed' | 'paused';
}

export class EventReplayer {
  private progress: ReplayProgress | null = null;

  async replay<TState>(
    options: ReplayOptions,
    handler: (state: TState, event: StoredEvent) => TState,
    initialState: TState,
    onComplete: (finalState: TState, version: number) => Promise<void>
  ): Promise<ReplayProgress> {
    const batchSize = options.batchSize ?? 1000;
    const checkpointEvery = options.checkpointEvery ?? 10_000;

    // チェックポイントから再開
    const checkpoint = await this.loadCheckpoint(options.projectionName);
    const startFrom = checkpoint?.lastProcessedVersion ?? (options.fromVersion ?? 0);

    const totalEvents = await this.countEvents(options.projectionName, startFrom, options.toVersion);

    this.progress = {
      projectionName: options.projectionName,
      totalEvents,
      processedEvents: checkpoint?.processedCount ?? 0,
      lastCheckpoint: startFrom,
      startedAt: new Date(),
      status: 'running',
    };

    logger.info({ ...this.progress }, 'Event replay started');

    let state = checkpoint?.state as TState ?? initialState;
    let currentVersion = startFrom;
    let processedCount = 0;

    while (true) {
      const events = await this.loadEvents(options.projectionName, currentVersion, batchSize, options.toVersion);

      if (events.length === 0) break;

      // バッチをメモリ上で処理
      for (const event of events) {
        state = handler(state, event);
        processedCount++;
        currentVersion = event.version;

        // チェックポイント保存
        if (processedCount % checkpointEvery === 0) {
          await this.saveCheckpoint(options.projectionName, {
            lastProcessedVersion: currentVersion,
            processedCount,
            state,
          });

          this.progress!.processedEvents = processedCount;
          this.progress!.lastCheckpoint = currentVersion;

          const elapsed = Date.now() - this.progress!.startedAt.getTime();
          const rate = processedCount / (elapsed / 1000); // events/sec
          const remaining = totalEvents - processedCount;
          this.progress!.estimatedCompletionAt = new Date(Date.now() + (remaining / rate) * 1000);

          logger.info(
            { processed: processedCount, total: totalEvents, rate: Math.round(rate), checkpoint: currentVersion },
            'Replay checkpoint saved'
          );
        }
      }

      if (events.length < batchSize) break; // 最後のバッチ
    }

    // リプレイ完了: 最終状態を書き込み
    await onComplete(state, currentVersion);

    // チェックポイント削除
    await this.deleteCheckpoint(options.projectionName);

    this.progress!.status = 'completed';
    this.progress!.processedEvents = processedCount;

    logger.info({ projectionName: options.projectionName, processedCount }, 'Event replay completed');

    return this.progress!;
  }

  private async loadEvents(projectionName: string, afterVersion: number, limit: number, maxVersion?: number): Promise<StoredEvent[]> {
    return prisma.eventStore.findMany({
      where: {
        aggregateType: projectionName,
        version: {
          gt: afterVersion,
          ...(maxVersion ? { lte: maxVersion } : {}),
        },
      },
      orderBy: { version: 'asc' },
      take: limit,
    });
  }

  private async countEvents(projectionName: string, afterVersion: number, maxVersion?: number): Promise<number> {
    return prisma.eventStore.count({
      where: {
        aggregateType: projectionName,
        version: { gt: afterVersion, ...(maxVersion ? { lte: maxVersion } : {}) },
      },
    });
  }

  private async saveCheckpoint(name: string, data: { lastProcessedVersion: number; processedCount: number; state: unknown }): Promise<void> {
    await redis.set(`replay:checkpoint:${name}`, JSON.stringify(data), { EX: 86400 });
  }

  private async loadCheckpoint(name: string): Promise<{ lastProcessedVersion: number; processedCount: number; state: unknown } | null> {
    const raw = await redis.get(`replay:checkpoint:${name}`);
    return raw ? JSON.parse(raw) : null;
  }

  private async deleteCheckpoint(name: string): Promise<void> {
    await redis.del(`replay:checkpoint:${name}`);
  }

  getProgress(): ReplayProgress | null {
    return this.progress;
  }
}
```

```typescript
// src/eventsourcing/replay/revenueProjectionReplayer.ts — 売上集計プロジェクションのリプレイ

export async function replayRevenueProjection(): Promise<void> {
  const replayer = new EventReplayer();

  interface RevenueState {
    dailyRevenue: Record<string, number>;   // 日付 → 売上
    productRevenue: Record<string, number>; // 商品ID → 売上
  }

  await replayer.replay<RevenueState>(
    {
      projectionName: 'order',
      batchSize: 1000,
      checkpointEvery: 5_000,
    },
    // ハンドラー: イベントを処理して状態を更新
    (state, event) => {
      const payload = JSON.parse(event.payload);

      if (event.eventType === 'OrderCompleted') {
        const date = new Date(payload.completedAt).toISOString().slice(0, 10);
        state.dailyRevenue[date] = (state.dailyRevenue[date] ?? 0) + payload.totalAmount;

        for (const item of payload.items) {
          state.productRevenue[item.productId] =
            (state.productRevenue[item.productId] ?? 0) + item.price * item.quantity;
        }
      }

      if (event.eventType === 'OrderRefunded') {
        const date = new Date(payload.refundedAt).toISOString().slice(0, 10);
        state.dailyRevenue[date] = (state.dailyRevenue[date] ?? 0) - payload.refundAmount;
      }

      return state;
    },
    // 初期状態
    { dailyRevenue: {}, productRevenue: {} },
    // 完了時: プロジェクションテーブルを更新
    async (finalState, lastVersion) => {
      await prisma.$transaction(async (tx) => {
        // 既存の集計をアップサートで置き換え（冪等）
        for (const [date, revenue] of Object.entries(finalState.dailyRevenue)) {
          await tx.$executeRaw`
            INSERT INTO daily_revenue (date, revenue, last_event_version)
            VALUES (${date}, ${revenue}, ${lastVersion})
            ON CONFLICT (date) DO UPDATE SET revenue = EXCLUDED.revenue, last_event_version = EXCLUDED.last_event_version
          `;
        }
      });

      logger.info({ lastVersion, dates: Object.keys(finalState.dailyRevenue).length }, 'Revenue projection updated');
    }
  );
}

// 管理者API: リプレイを起動
router.post('/api/admin/projections/:name/replay', requireAdmin, async (req, res) => {
  const replayer = new EventReplayer();

  // バックグラウンドでリプレイ開始
  replayRevenueProjection().catch(err => logger.error({ err }, 'Replay failed'));

  res.json({ message: 'Replay started', progress: replayer.getProgress() });
});

router.get('/api/admin/projections/:name/replay/progress', requireAdmin, async (req, res) => {
  const replayer = new EventReplayer();
  res.json(replayer.getProgress() ?? { status: 'not-running' });
});
```

---

## まとめ

Claude Codeでイベントリプレイを設計する：

1. **CLAUDE.md** にページング（1バッチ1000件）でメモリ消費を制限・チェックポイントを定期保存して障害時に途中から再開・`ON CONFLICT DO UPDATE`で冪等な集計更新を明記
2. **チェックポイント（Redis）** で長時間リプレイのレジリエンス確保——100万件を処理中にクラッシュしても、最後のチェックポイント（例: 50万件目）から再開できる
3. **進捗推定（rate計算）** で「あと何分で完了するか」をリアルタイム表示——管理者が「今すぐデプロイしてOKか」を判断するために必要な情報
4. **`ON CONFLICT DO UPDATE`（アップサート）** でプロジェクションの冪等更新——同じイベントを2回リプレイしても売上の二重計上にならない。リプレイを何度実行しても同じ結果になる

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
