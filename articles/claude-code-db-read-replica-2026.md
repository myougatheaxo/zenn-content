---
title: "Claude CodeでDBリードレプリカを設計する：読み書き分離・負荷分散・Prisma統合"
emoji: "🗄️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "prisma"]
published: true
---

## はじめに

DBの読み込みが遅い——リードレプリカで読み書きを分離し、SELECT系クエリをレプリカに振り分けてプライマリの負荷を下げる。Prisma + PgBouncerパターンをClaude Codeに設計させる。

---

## CLAUDE.mdにリードレプリカ設計ルールを書く

```markdown
## DBリードレプリカ設計ルール

### ルーティング方針
- 書き込み（INSERT/UPDATE/DELETE）: primaryのみ
- 読み込み（SELECT）: replicaにルーティング
- トランザクション内: 全てprimaryを使用
- 一貫性が必要な読み込み: primaryを明示的に指定

### レプリカ遅延対策
- 書き込み後の即時読み込みはprimaryから
- キャッシュ（Redis）で遅延を補完
- レプリカ遅延監視（max_lag: 10秒）

### フェイルオーバー
- レプリカ障害時はprimaryにフォールバック
- ヘルスチェックで不健全なレプリカを除外
```

---

## リードレプリカ実装の生成

```
PostgreSQLリードレプリカの読み書き分離を設計してください。

要件：
- Prismaで複数DB接続管理
- 自動ルーティング（写込み→primary、読み込み→replica）
- レプリカ遅延対策
- ヘルスチェック

生成ファイル: src/db/
```

---

## 生成されるリードレプリカ実装

```typescript
// src/db/clients.ts

import { PrismaClient } from '@prisma/client';

// プライマリDB（書き込み専用）
export const primaryDb = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_URL! },
  },
  log: ['warn', 'error'],
});

// リードレプリカ（読み込み専用）
export const replicaDb = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_REPLICA_URL! },
  },
  log: ['warn', 'error'],
});

// ヘルスチェック付きレプリカプール
class ReplicaPool {
  private replicas: PrismaClient[];
  private healthStatus: Map<PrismaClient, boolean> = new Map();
  private currentIndex = 0;

  constructor(replicaUrls: string[]) {
    this.replicas = replicaUrls.map(url => new PrismaClient({
      datasources: { db: { url } },
    }));
    this.replicas.forEach(r => this.healthStatus.set(r, true));

    // 10秒ごとにヘルスチェック
    setInterval(() => this.checkHealth(), 10_000);
  }

  private async checkHealth(): Promise<void> {
    await Promise.all(
      this.replicas.map(async (replica) => {
        try {
          await replica.$queryRaw`SELECT 1`;
          this.healthStatus.set(replica, true);
        } catch {
          this.healthStatus.set(replica, false);
          logger.warn('Replica unhealthy, removed from pool');
        }
      })
    );
  }

  // ラウンドロビンで健全なレプリカを返す
  getHealthyReplica(): PrismaClient {
    const healthy = this.replicas.filter(r => this.healthStatus.get(r));
    if (healthy.length === 0) {
      logger.warn('All replicas unhealthy, falling back to primary');
      return primaryDb; // フォールバック
    }
    const replica = healthy[this.currentIndex % healthy.length];
    this.currentIndex++;
    return replica;
  }
}

const replicaUrls = (process.env.DATABASE_REPLICA_URLS ?? process.env.DATABASE_REPLICA_URL ?? '')
  .split(',')
  .filter(Boolean);

export const replicaPool = new ReplicaPool(replicaUrls);
```

```typescript
// src/db/router.ts — 自動ルーティング

// コンテキストで現在のDB接続を管理（AsyncLocalStorage）
import { AsyncLocalStorage } from 'async_hooks';

const dbContext = new AsyncLocalStorage<{ db: PrismaClient; forceReplica?: boolean }>();

// 通常のDB取得（自動ルーティング）
export function getDb(): PrismaClient {
  const ctx = dbContext.getStore();
  return ctx?.db ?? primaryDb; // デフォルトはprimary
}

// 読み込み専用DB取得（replica優先）
export function getReadDb(): PrismaClient {
  const ctx = dbContext.getStore();
  if (ctx?.db) return ctx.db; // トランザクション中はそのDBを使用

  return replicaPool.getHealthyReplica();
}

// 書き込み後に一定時間primaryから読む（レプリカ遅延対策）
export async function withPostWriteConsistency<T>(
  fn: () => Promise<T>,
  durationMs = 2000
): Promise<T> {
  return dbContext.run({ db: primaryDb }, fn);
}

// トランザクション（全てprimary）
export async function withTransaction<T>(
  fn: (tx: Prisma.TransactionClient) => Promise<T>
): Promise<T> {
  return primaryDb.$transaction(async (tx) => {
    return dbContext.run({ db: tx as any }, () => fn(tx));
  });
}
```

```typescript
// src/repositories/userRepository.ts — 実際の使用例

export class UserRepository {
  // 書き込み: primaryへ
  async create(data: CreateUserInput): Promise<User> {
    const user = await primaryDb.user.create({ data });

    // 書き込み後2秒はprimaryから読む（レプリカ遅延対策）
    await withPostWriteConsistency(async () => {
      await redis.setex(`user:fresh:${user.id}`, 2, '1'); // 2秒間フレッシュフラグ
    });

    return user;
  }

  // 読み込み: replicaへ（レイテンシ削減）
  async findById(id: string): Promise<User | null> {
    // 直近で書き込みがあった場合はprimaryから
    const isFresh = await redis.get(`user:fresh:${id}`);
    const db = isFresh ? primaryDb : getReadDb();

    return db.user.findUnique({ where: { id } });
  }

  // 一覧取得（常にreplica）
  async findMany(params: FindUsersParams): Promise<User[]> {
    return getReadDb().user.findMany({
      where: params.where,
      orderBy: params.orderBy,
      skip: params.skip,
      take: params.take,
    });
  }
}
```

---

## レプリカ遅延監視

```typescript
// src/db/monitoring.ts

// レプリカ遅延を定期的にチェック
export async function checkReplicaLag(): Promise<{ lagSeconds: number; healthy: boolean }> {
  const result = await replicaDb.$queryRaw<[{ lag: number }]>`
    SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag
  `;

  const lagSeconds = result[0]?.lag ?? 0;
  const healthy = lagSeconds < 10; // 10秒以上の遅延は警告

  if (!healthy) {
    logger.warn({ lagSeconds }, 'Replica lag is high');
    await sendAlert('replica_lag_high', { lagSeconds });
  }

  return { lagSeconds, healthy };
}

// Prometheusメトリクスとして公開
const replicaLagGauge = new Gauge({
  name: 'db_replica_lag_seconds',
  help: 'PostgreSQL replica lag in seconds',
});

setInterval(async () => {
  const { lagSeconds } = await checkReplicaLag();
  replicaLagGauge.set(lagSeconds);
}, 30_000);
```

---

## まとめ

Claude CodeでDBリードレプリカを設計する：

1. **CLAUDE.md** に書き込み→primary・読み込み→replica・トランザクション内はprimaryを明記
2. **AsyncLocalStorage** でコンテキストベースのDB自動ルーティング
3. **書き込み後2秒はprimaryフラグ** をRedisに設定してレプリカ遅延を回避
4. **ReplicaPool** でヘルスチェック付きラウンドロビン→レプリカ障害時にprimaryフォールバック

---

*DBリードレプリカ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
