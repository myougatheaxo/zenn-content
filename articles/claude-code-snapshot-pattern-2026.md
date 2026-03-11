---
title: "Claude Codeでスナップショットパターンを設計する：集約の状態保存・Event Sourcing高速化・ポイントインタイム復元"
emoji: "📸"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-17 20:00"
---

## はじめに

「Event Sourcingで全イベントを再生して状態を再構築するのが10万イベント分あって遅い」——スナップショットパターンで最新状態を定期保存し、再構築を直近のスナップショットからのイベント再生だけに短縮する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにスナップショット設計ルールを書く

```markdown
## スナップショットパターン設計ルール

### スナップショット作成タイミング
- イベント数が閾値（例: 100イベント）を超えた時
- 定期的（例: 1日1回）に最新状態をスナップショット
- 手動トリガー（管理者が指定した集約を即座にスナップショット）

### スナップショットの内容
- 集約ID・バージョン番号・状態のJSON・作成日時
- バージョン番号 = スナップショット時点でのイベント数
- 状態再構築時: 最新スナップショット → それ以降のイベントだけを再生

### 古いスナップショットの管理
- 最新の3世代のみ保持（それ以前は削除）
- イベントは削除しない（監査ログとして永続保持）
```

---

## スナップショット実装の生成

```
スナップショットパターンを設計してください。

要件：
- 自動スナップショット作成（イベント数閾値）
- スナップショットからの高速状態再構築
- ポイントインタイム復元
- スナップショット世代管理

生成ファイル: src/eventsourcing/snapshot/
```

---

## 生成されるスナップショット実装

```typescript
// src/eventsourcing/snapshot/snapshotStore.ts — スナップショットストア

export interface Snapshot<TState> {
  aggregateId: string;
  aggregateType: string;
  version: number;        // このスナップショット時点のイベント数
  state: TState;
  createdAt: Date;
}

export class SnapshotStore {
  private readonly GENERATIONS_TO_KEEP = 3;

  async save<TState>(snapshot: Snapshot<TState>): Promise<void> {
    await prisma.$transaction(async (tx) => {
      // 新しいスナップショットを保存
      await tx.aggregateSnapshot.create({
        data: {
          aggregateId: snapshot.aggregateId,
          aggregateType: snapshot.aggregateType,
          version: snapshot.version,
          state: JSON.stringify(snapshot.state),
          createdAt: snapshot.createdAt,
        },
      });

      // 古い世代を削除（最新3世代のみ保持）
      const oldSnapshots = await tx.aggregateSnapshot.findMany({
        where: {
          aggregateId: snapshot.aggregateId,
          aggregateType: snapshot.aggregateType,
        },
        orderBy: { version: 'desc' },
        skip: this.GENERATIONS_TO_KEEP,
      });

      if (oldSnapshots.length > 0) {
        await tx.aggregateSnapshot.deleteMany({
          where: { id: { in: oldSnapshots.map(s => s.id) } },
        });
      }
    });

    logger.info(
      { aggregateId: snapshot.aggregateId, version: snapshot.version },
      'Snapshot saved'
    );
  }

  async getLatest<TState>(aggregateId: string, aggregateType: string): Promise<Snapshot<TState> | null> {
    const row = await prisma.aggregateSnapshot.findFirst({
      where: { aggregateId, aggregateType },
      orderBy: { version: 'desc' },
    });

    if (!row) return null;

    return {
      aggregateId: row.aggregateId,
      aggregateType: row.aggregateType,
      version: row.version,
      state: JSON.parse(row.state) as TState,
      createdAt: row.createdAt,
    };
  }

  // ポイントインタイム: 特定バージョン以前の最新スナップショット
  async getAtVersion<TState>(
    aggregateId: string,
    aggregateType: string,
    targetVersion: number
  ): Promise<Snapshot<TState> | null> {
    const row = await prisma.aggregateSnapshot.findFirst({
      where: { aggregateId, aggregateType, version: { lte: targetVersion } },
      orderBy: { version: 'desc' },
    });

    if (!row) return null;
    return { aggregateId, aggregateType, version: row.version, state: JSON.parse(row.state), createdAt: row.createdAt };
  }
}
```

```typescript
// src/eventsourcing/snapshot/snapshotRepository.ts — スナップショット付きリポジトリ

export abstract class SnapshotRepository<TAggregate, TState, TEvent> {
  private readonly SNAPSHOT_THRESHOLD = 100; // 100イベント毎にスナップショット
  private readonly snapshots = new SnapshotStore();

  // 集約を高速ロード（スナップショット + 差分イベント）
  async load(aggregateId: string): Promise<TAggregate> {
    const snapshot = await this.snapshots.getLatest<TState>(aggregateId, this.aggregateType);

    let state: TState;
    let fromVersion: number;

    if (snapshot) {
      state = snapshot.state;
      fromVersion = snapshot.version;
      logger.debug({ aggregateId, snapshotVersion: fromVersion }, 'Loading from snapshot');
    } else {
      state = this.initialState();
      fromVersion = 0;
    }

    // スナップショット以降のイベントのみ再生
    const events = await this.loadEventsSince(aggregateId, fromVersion);

    for (const event of events) {
      state = this.applyEvent(state, event);
    }

    logger.debug(
      { aggregateId, snapshotVersion: fromVersion, replayedEvents: events.length },
      'Aggregate loaded'
    );

    return this.createAggregate(aggregateId, state, fromVersion + events.length);
  }

  // 保存時にスナップショット閾値チェック
  async save(aggregate: TAggregate, newEvents: TEvent[]): Promise<void> {
    const aggregateId = this.getAggregateId(aggregate);
    const currentVersion = this.getVersion(aggregate);

    await prisma.$transaction(async (tx) => {
      // イベントを保存
      await this.saveEvents(tx, aggregateId, currentVersion, newEvents);

      // スナップショット閾値チェック
      const newVersion = currentVersion + newEvents.length;
      if (newVersion % this.SNAPSHOT_THRESHOLD === 0) {
        const state = this.getState(aggregate);
        const finalState = newEvents.reduce(this.applyEvent.bind(this), state);

        await this.snapshots.save({
          aggregateId,
          aggregateType: this.aggregateType,
          version: newVersion,
          state: finalState,
          createdAt: new Date(),
        });

        logger.info({ aggregateId, version: newVersion }, 'Auto-snapshot created');
      }
    });
  }

  // ポイントインタイム復元（過去の状態を再現）
  async loadAt(aggregateId: string, targetVersion: number): Promise<TAggregate> {
    const snapshot = await this.snapshots.getAtVersion<TState>(
      aggregateId, this.aggregateType, targetVersion
    );

    let state = snapshot ? snapshot.state : this.initialState();
    const fromVersion = snapshot?.version ?? 0;

    // targetVersionまでのイベントを再生
    const events = await this.loadEventsRange(aggregateId, fromVersion, targetVersion);

    for (const event of events) {
      state = this.applyEvent(state, event);
    }

    return this.createAggregate(aggregateId, state, targetVersion);
  }

  // サブクラスで実装
  protected abstract readonly aggregateType: string;
  protected abstract initialState(): TState;
  protected abstract applyEvent(state: TState, event: TEvent): TState;
  protected abstract createAggregate(id: string, state: TState, version: number): TAggregate;
  protected abstract getAggregateId(aggregate: TAggregate): string;
  protected abstract getVersion(aggregate: TAggregate): number;
  protected abstract getState(aggregate: TAggregate): TState;
  protected abstract loadEventsSince(id: string, fromVersion: number): Promise<TEvent[]>;
  protected abstract loadEventsRange(id: string, from: number, to: number): Promise<TEvent[]>;
  protected abstract saveEvents(tx: any, id: string, version: number, events: TEvent[]): Promise<void>;
}

// 管理者向けスナップショット作成API
router.post('/api/admin/aggregates/:type/:id/snapshot', requireAdmin, async (req, res) => {
  const repo = getRepository(req.params.type);
  const aggregate = await repo.load(req.params.id);

  const snapshot: Snapshot<any> = {
    aggregateId: req.params.id,
    aggregateType: req.params.type,
    version: aggregate.version,
    state: aggregate.state,
    createdAt: new Date(),
  };

  const store = new SnapshotStore();
  await store.save(snapshot);

  res.json({ message: 'Snapshot created', version: snapshot.version });
});
```

---

## まとめ

Claude Codeでスナップショットパターンを設計する：

1. **CLAUDE.md** に100イベント毎に自動スナップショット・スナップショット以降のイベントのみ再生・最新3世代保持・イベントは削除しないを明記
2. **スナップショット + 差分再生** で10万イベントの集約でも「最新スナップショット（例: 99900番目）+ 最後の100イベント」だけ再生——再構築時間をO(n)からO(閾値)に削減
3. **ポイントインタイム復元（`loadAt`）** で過去の任意バージョンの状態を再現——「6ヶ月前のアカウント状態を確認して欲しい」といった監査要求に対応
4. **古い世代の自動削除** で最新3世代のみ保持——スナップショットが際限なく増えてストレージを消費することを防止

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
