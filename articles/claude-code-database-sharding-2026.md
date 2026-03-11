---
title: "Claude Codeでデータベースシャーディングを設計する：水平分割・ルーティング・クロスシャードクエリ"
emoji: "🗃️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
---

## はじめに

1つのDBサーバーが書き込みのボトルネックになり始めた。シャーディングで複数のDBサーバーにデータを分散させる。ただしクロスシャードクエリは使えないため、シャードキーの設計が最も重要だ。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにシャーディング設計ルールを書く

```markdown
## データベースシャーディング設計ルール

### シャードキー選択基準
- カーディナリティが高い（ユーザーID、テナントID等）
- 均等分散される（ホットシャードを防ぐ）
- クエリパターンと一致する（シャードキーなしのクエリを避ける）
- 変更しない（一度決めたら変更困難）

### シャーディング方式
- ハッシュシャーディング: 均等分散（範囲クエリに弱い）
- レンジシャーディング: 範囲クエリに強い（ホットシャードリスク）
- ディレクトリシャーディング: 柔軟だがルーティングDBが必要

### クロスシャード制約
- JOINはアプリケーション層で行う（シャードをまたぐJOIN禁止）
- COUNT(*)/SUM等の集計はMapReduceパターン
- グローバルユニーク性はULIDまたはUUIDv7で保証
```

---

## シャーディングシステムの生成

```
テナントIDベースのシャーディングシステムを設計してください。

要件：
- テナントIDでシャード番号を決定（ハッシュ）
- シャードプールの管理
- クロスシャードクエリの回避パターン
- シャードの追加（リバランス）

生成ファイル: src/sharding/
```

---

## 生成されるシャーディング実装

```typescript
// src/sharding/shardRouter.ts
import crypto from 'crypto';

interface ShardConfig {
  id: number;
  host: string;
  port: number;
  database: string;
  replicaHost?: string; // 読み取りレプリカ
}

// シャード設定（本番ではDBから読み込む）
const SHARD_CONFIGS: ShardConfig[] = [
  { id: 0, host: 'db-shard-0.internal', port: 5432, database: 'appdb_0' },
  { id: 1, host: 'db-shard-1.internal', port: 5432, database: 'appdb_1' },
  { id: 2, host: 'db-shard-2.internal', port: 5432, database: 'appdb_2' },
  { id: 3, host: 'db-shard-3.internal', port: 5432, database: 'appdb_3' },
];

const SHARD_COUNT = SHARD_CONFIGS.length;

export class ShardRouter {
  // テナントIDからシャード番号を決定（一貫性のあるハッシュ）
  getShardId(tenantId: string): number {
    const hash = crypto.createHash('sha256').update(tenantId).digest('hex');
    const hashInt = parseInt(hash.slice(0, 8), 16);
    return hashInt % SHARD_COUNT;
  }

  getShardConfig(tenantId: string): ShardConfig {
    const shardId = this.getShardId(tenantId);
    return SHARD_CONFIGS[shardId];
  }

  // 全シャードの設定を返す（集計クエリ用）
  getAllShards(): ShardConfig[] {
    return SHARD_CONFIGS;
  }
}

export const shardRouter = new ShardRouter();
```

```typescript
// src/sharding/shardedPrisma.ts
import { PrismaClient } from '@prisma/client';

// シャードごとのPrismaクライアントをキャッシュ
const shardClients = new Map<number, PrismaClient>();

export function getPrismaForTenant(tenantId: string): PrismaClient {
  const shardConfig = shardRouter.getShardConfig(tenantId);

  if (!shardClients.has(shardConfig.id)) {
    const client = new PrismaClient({
      datasources: {
        db: {
          url: `postgresql://${process.env.DB_USER}:${process.env.DB_PASSWORD}@${shardConfig.host}:${shardConfig.port}/${shardConfig.database}`,
        },
      },
    });
    shardClients.set(shardConfig.id, client);
  }

  return shardClients.get(shardConfig.id)!;
}

// シャードを意識したリポジトリ
export class ShardedOrderRepository {
  async create(tenantId: string, data: CreateOrderData): Promise<Order> {
    const prisma = getPrismaForTenant(tenantId);
    return prisma.order.create({
      data: {
        ...data,
        tenantId, // シャードキーをデータに含める
      },
    });
  }

  async findByTenant(tenantId: string, options: { take?: number; skip?: number } = {}): Promise<Order[]> {
    const prisma = getPrismaForTenant(tenantId);
    return prisma.order.findMany({
      where: { tenantId }, // シャードキーで絞り込む（シャードをまたがない）
      ...options,
      orderBy: { createdAt: 'desc' },
    });
  }

  async findById(tenantId: string, orderId: string): Promise<Order | null> {
    const prisma = getPrismaForTenant(tenantId);
    return prisma.order.findFirst({
      where: {
        id: orderId,
        tenantId, // テナントIDで絞り込み（誤ったシャードを検索しない）
      },
    });
  }
}
```

---

## クロスシャード集計（MapReduce）

```typescript
// src/sharding/crossShardQuery.ts

// 全シャードにわたる集計（MapReduceパターン）
export async function getGlobalOrderStats(): Promise<OrderStats> {
  // Map: 各シャードで並列集計
  const shardResults = await Promise.all(
    shardRouter.getAllShards().map(async (shard) => {
      const shardId = shard.id;
      const prisma = shardClients.get(shardId) ?? new PrismaClient({ datasources: { db: { url: buildUrl(shard) } } });

      const [count, revenue] = await Promise.all([
        prisma.order.count({ where: { status: 'completed' } }),
        prisma.order.aggregate({
          where: { status: 'completed' },
          _sum: { totalCents: true },
        }),
      ]);

      return {
        shardId,
        completedOrders: count,
        totalRevenueCents: revenue._sum.totalCents ?? 0,
      };
    })
  );

  // Reduce: 全シャードの結果を集約
  return shardResults.reduce(
    (acc, result) => ({
      completedOrders: acc.completedOrders + result.completedOrders,
      totalRevenueCents: acc.totalRevenueCents + result.totalRevenueCents,
    }),
    { completedOrders: 0, totalRevenueCents: 0 }
  );
}
```

---

## テナントをまたいだ検索（Elasticsearchにオフロード）

```typescript
// src/search/globalSearch.ts
// シャードをまたぐ検索はElasticsearchに委ねる

export async function searchOrdersGlobal(query: string): Promise<SearchResult[]> {
  // Elasticsearchで全テナントを検索（シャード非依存）
  const result = await elasticsearch.search({
    index: 'orders',
    body: {
      query: {
        multi_match: {
          query,
          fields: ['orderId', 'customerName', 'productName'],
        },
      },
      size: 20,
    },
  });

  return result.hits.hits.map(hit => ({
    orderId: hit._source.orderId,
    tenantId: hit._source.tenantId,
    customerName: hit._source.customerName,
    score: hit._score,
  }));
}

// DB書き込み時にElasticsearchにも同期
prisma.$use(async (params, next) => {
  const result = await next(params);
  if (params.model === 'Order' && params.action === 'create') {
    setImmediate(() =>
      elasticsearch.index({
        index: 'orders',
        id: result.id,
        body: {
          orderId: result.id,
          tenantId: result.tenantId,
          customerName: result.customerName,
        },
      }).catch(err => logger.error({ err }, 'ES sync failed'))
    );
  }
  return result;
});
```

---

## まとめ

Claude Codeでデータベースシャーディングを設計する：

1. **CLAUDE.md** にシャードキー選択基準・クロスシャードJOIN禁止・集計方針を明記
2. **テナントIDでハッシュ** してシャード番号を決定（一貫性のある分散）
3. **全クエリにシャードキー** を含める（シャードをまたがないよう強制）
4. **クロスシャード集計** はMapReduceパターン、検索はElasticsearchにオフロード

---

*シャーディング設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
