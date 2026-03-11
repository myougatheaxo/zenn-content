---
title: "Claude Codeでリーダー選出を設計する：Redis SET NX・リース更新・フェイルオーバー"
emoji: "👑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-18 09:00"
---

## はじめに

「cronジョブを複数台で動かしたら同じ処理が重複実行された」——Redisを使ったリーダー選出で、複数のインスタンスの中から1つだけがリーダーとして動き、リーダーが落ちたら自動でフェイルオーバーする設計をClaude Codeに生成させる。

---

## CLAUDE.mdにリーダー選出設計ルールを書く

```markdown
## リーダー選出設計ルール

### リース（Lease）
- リーダーはRedis SET NXでリースキーを取得（TTL: 30秒）
- リーダーはTTLが切れる前（例: 10秒ごと）にリースを更新
- リーダーが落ちてTTLが切れると、他のインスタンスがリースを取得

### フェイルオーバー
- TTL切れ後にフォロワーが競争してリースを取得
- 取得できたインスタンスが新リーダー
- フェイルオーバー時間 = リースTTL（最大30秒）

### フェンシング
- リーダーIDをメッセージに付与（誰がリーダーとして実行したか記録）
- 脳分裂（Split Brain）対策: 操作にエポック番号を付与
```

---

## リーダー選出実装の生成

```
リーダー選出システムを設計してください。

要件：
- Redis SET NXでのリース取得
- 自動リース更新
- フェイルオーバー検出
- フェンシングトークン

生成ファイル: src/distributed/leaderElection/
```

---

## 生成されるリーダー選出実装

```typescript
// src/distributed/leaderElection/leaderElection.ts — リーダー選出

export interface LeaderElectionOptions {
  resourceName: string;   // 選出対象のリソース名（例: 'cron-scheduler'）
  instanceId: string;     // このインスタンスのID（例: Pod名 + ULIDなど）
  leaseTtlMs: number;     // リースTTL（例: 30秒）
  renewIntervalMs: number;// リース更新間隔（TTLの1/3程度が安全）
}

export class LeaderElection {
  private isLeader = false;
  private leaseEpoch = 0;   // フェンシング用エポック番号
  private renewTimer: NodeJS.Timeout | null = null;
  private onBecomeLeaderCallbacks: Array<() => Promise<void>> = [];
  private onLoseLeadershipCallbacks: Array<() => Promise<void>> = [];

  constructor(private readonly opts: LeaderElectionOptions) {}

  // リーダー選出を開始（バックグラウンドでリース取得を試み続ける）
  async start(): Promise<void> {
    logger.info({ instanceId: this.opts.instanceId, resource: this.opts.resourceName }, 'Starting leader election');

    while (true) {
      if (!this.isLeader) {
        const acquired = await this.tryAcquireLease();

        if (acquired) {
          this.isLeader = true;
          this.leaseEpoch++;
          logger.info(
            { instanceId: this.opts.instanceId, epoch: this.leaseEpoch },
            'Became leader'
          );

          // リース更新タイマー開始
          this.startRenewing();

          // リーダー就任コールバック
          for (const cb of this.onBecomeLeaderCallbacks) {
            await cb().catch(err => logger.error({ err }, 'onBecomeLeader callback error'));
          }
        }
      }

      // フォロワーは定期的にリース取得を試みる
      await sleep(this.opts.renewIntervalMs);
    }
  }

  private async tryAcquireLease(): Promise<boolean> {
    const key = `leader:${this.opts.resourceName}`;
    const acquired = await redis.set(
      key,
      JSON.stringify({ instanceId: this.opts.instanceId, acquiredAt: Date.now() }),
      { NX: true, PX: this.opts.leaseTtlMs }
    );
    return acquired !== null;
  }

  private startRenewing(): void {
    this.renewTimer = setInterval(async () => {
      const renewed = await this.renewLease();

      if (!renewed) {
        // リース更新失敗（ネットワーク断など）→ リーダー資格を放棄
        logger.warn({ instanceId: this.opts.instanceId }, 'Lease renewal failed — yielding leadership');
        await this.yieldLeadership();
      }
    }, this.opts.renewIntervalMs);
  }

  private async renewLease(): Promise<boolean> {
    const key = `leader:${this.opts.resourceName}`;

    // LuaスクリプトでアトミックにTTLをリセット（自分のリースのみ）
    const result = await redis.eval(
      `
        local current = redis.call('GET', KEYS[1])
        if current == false then return 0 end
        local data = cjson.decode(current)
        if data['instanceId'] ~= ARGV[1] then return 0 end
        redis.call('PEXPIRE', KEYS[1], ARGV[2])
        return 1
      `,
      [key],
      [this.opts.instanceId, this.opts.leaseTtlMs.toString()]
    );

    return result === 1;
  }

  async yieldLeadership(): Promise<void> {
    if (!this.isLeader) return;

    this.isLeader = false;

    if (this.renewTimer) {
      clearInterval(this.renewTimer);
      this.renewTimer = null;
    }

    // リーダーシップ喪失コールバック
    for (const cb of this.onLoseLeadershipCallbacks) {
      await cb().catch(err => logger.error({ err }, 'onLoseLeadership callback error'));
    }

    logger.info({ instanceId: this.opts.instanceId }, 'Leadership yielded');
  }

  // フェンシングトークン（エポック番号）を取得
  // リーダーが実行する操作にこのトークンを付与し、古いリーダーの操作を排除
  getFencingToken(): { instanceId: string; epoch: number } | null {
    if (!this.isLeader) return null;
    return { instanceId: this.opts.instanceId, epoch: this.leaseEpoch };
  }

  get currentlyLeader(): boolean {
    return this.isLeader;
  }

  onBecomeLeader(callback: () => Promise<void>): void {
    this.onBecomeLeaderCallbacks.push(callback);
  }

  onLoseLeadership(callback: () => Promise<void>): void {
    this.onLoseLeadershipCallbacks.push(callback);
  }
}
```

```typescript
// src/distributed/leaderElection/schedulerLeader.ts — リーダー限定のcronスケジューラー

const election = new LeaderElection({
  resourceName: 'cron-scheduler',
  instanceId: `${process.env.POD_NAME}-${ulid()}`,
  leaseTtlMs: 30_000,    // 30秒
  renewIntervalMs: 10_000, // 10秒ごとに更新
});

// リーダーになった時だけスケジューラーを起動
election.onBecomeLeader(async () => {
  logger.info('Starting cron scheduler as leader');
  startScheduler();
});

// リーダーシップを失ったらスケジューラーを停止
election.onLoseLeadership(async () => {
  logger.info('Stopping cron scheduler — lost leadership');
  stopScheduler();
});

election.start().catch(err => {
  logger.error({ err }, 'Leader election failed');
  process.exit(1);
});

// フェンシングトークンを使ったDB操作（古いリーダーの操作を排除）
async function runScheduledJob(): Promise<void> {
  const token = election.getFencingToken();
  if (!token) {
    logger.debug('Not leader — skipping scheduled job');
    return;
  }

  // フェンシングトークンをJob実行記録に含める
  await prisma.scheduledJobRun.create({
    data: {
      jobId: 'daily-report',
      leaderId: token.instanceId,
      epoch: token.epoch,
      startedAt: new Date(),
    },
  });

  // ジョブ実行...
}

// ヘルスチェック
router.get('/health/leader', (req, res) => {
  res.json({
    instanceId: process.env.POD_NAME,
    isLeader: election.currentlyLeader,
    fencingToken: election.getFencingToken(),
  });
});
```

---

## まとめ

Claude Codeでリーダー選出を設計する：

1. **CLAUDE.md** にRedis SET NXでリース取得・TTL30秒・10秒ごとに更新・TTL切れ後に他インスタンスが取得・フェンシングトークンでエポック管理を明記
2. **Luaスクリプトでアトミックなリース更新** で「自分のリースだけをリセット」——チェックしてから更新する間に他のインスタンスが取得してしまうTOCTOU問題を防止
3. **フェンシングトークン（エポック番号）** でネットワーク分断後の脳分裂対策——古いリーダーが`epoch=1`でDB操作しようとしても、新リーダーが`epoch=2`になっていれば古い操作を識別して拒否できる
4. **コールバックパターン（onBecomeLeader/onLoseLeadership）** でスケジューラーの起動/停止を宣言的に記述——リーダー状態の変化をイベントとして扱い、制御フローをクリーンに保つ

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
