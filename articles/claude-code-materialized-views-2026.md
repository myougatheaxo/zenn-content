---
title: "Claude CodeでPostgreSQLマテリアライズドビューを設計する：集計クエリ高速化・自動更新"
emoji: "🗃️"
type: "tech"
topics: ["claudecode", "typescript", "postgresql", "performance", "nodejs"]
published: true
published_at: "2026-03-15 20:00"
---

## はじめに

「ダッシュボードの集計クエリが3秒かかる」——PostgreSQLのマテリアライズドビューで重い集計を事前計算し、1ms未満で返す設計をClaude Codeに生成させる。

---

## CLAUDE.mdにマテリアライズドビュー設計ルールを書く

```markdown
## マテリアライズドビュー設計ルール

### 適用基準
- クエリが複数テーブルJOIN + GROUP BY + 集計関数を含む
- データが数秒〜数分遅れても許容できる（ダッシュボード・レポート系）
- 元データの更新頻度: 数分以内なら REFRESH CONCURRENTLY で対応

### 更新戦略
- REFRESH CONCURRENTLY: 読み取りロックなし（本番推奨）
- スケジュール: 5分ごと（BullMQまたはpg-cron）
- 更新後はアプリキャッシュも無効化

### インデックス設計
- マテリアライズドビューにもインデックスを張る
- よく絞り込むカラム（tenant_id, date等）に複合インデックス
```

---

## マテリアライズドビュー実装の生成

```
PostgreSQLマテリアライズドビューで集計クエリを高速化してください。

要件：
- 売上ダッシュボード用集計ビュー
- REFRESH CONCURRENTLY自動更新
- BullMQスケジューラー連携
- アプリキャッシュ無効化

生成ファイル: prisma/migrations/, src/db/materializedViews/
```

---

## 生成されるマテリアライズドビュー実装

```sql
-- prisma/migrations/20260315_create_materialized_views.sql

-- 日次売上集計ビュー（テナント×日付×カテゴリ）
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT
  u.tenant_id,
  DATE_TRUNC('day', o.created_at AT TIME ZONE 'Asia/Tokyo') AS sale_date,
  p.category,
  COUNT(DISTINCT o.id)                          AS order_count,
  COUNT(DISTINCT o.user_id)                     AS unique_buyers,
  SUM(oi.unit_price * oi.quantity)              AS revenue,
  AVG(oi.unit_price * oi.quantity)              AS avg_order_value,
  SUM(oi.quantity)                              AS total_units
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.status NOT IN ('cancelled', 'refunded')
  AND o.deleted_at IS NULL
GROUP BY u.tenant_id, DATE_TRUNC('day', o.created_at AT TIME ZONE 'Asia/Tokyo'), p.category
WITH DATA; -- 初回作成時にデータを即時投入

-- 高速化インデックス（ビューにも張れる）
CREATE INDEX idx_mv_daily_revenue_tenant_date
  ON mv_daily_revenue (tenant_id, sale_date DESC);

CREATE INDEX idx_mv_daily_revenue_category
  ON mv_daily_revenue (tenant_id, category, sale_date DESC);

-- ユーザーコホート集計ビュー（月次初回購入コホート）
CREATE MATERIALIZED VIEW mv_user_cohort AS
WITH first_orders AS (
  SELECT user_id, MIN(DATE_TRUNC('month', created_at)) AS cohort_month
  FROM orders
  WHERE status NOT IN ('cancelled', 'refunded') AND deleted_at IS NULL
  GROUP BY user_id
)
SELECT
  fo.cohort_month,
  DATE_TRUNC('month', o.created_at) AS activity_month,
  COUNT(DISTINCT fo.user_id)        AS cohort_size,
  COUNT(DISTINCT o.user_id)         AS active_users,
  ROUND(100.0 * COUNT(DISTINCT o.user_id) / COUNT(DISTINCT fo.user_id), 1) AS retention_rate
FROM first_orders fo
LEFT JOIN orders o ON o.user_id = fo.user_id
  AND o.status NOT IN ('cancelled', 'refunded')
  AND o.deleted_at IS NULL
GROUP BY fo.cohort_month, DATE_TRUNC('month', o.created_at)
WITH DATA;

CREATE UNIQUE INDEX idx_mv_cohort_unique
  ON mv_user_cohort (cohort_month, activity_month);
```

```typescript
// src/db/materializedViews/refresher.ts — ビュー更新

const MATERIALIZED_VIEWS = [
  { name: 'mv_daily_revenue',  priority: 'high' },
  { name: 'mv_user_cohort',    priority: 'low'  },
] as const;

export async function refreshView(viewName: string): Promise<{ durationMs: number }> {
  const start = Date.now();

  // CONCURRENTLY: 読み取りロックなし（更新中もSELECT可能）
  // UNIQUE インデックスが必要（CONCURRENTLYの前提条件）
  await prisma.$executeRawUnsafe(
    `REFRESH MATERIALIZED VIEW CONCURRENTLY ${viewName}`
  );

  const durationMs = Date.now() - start;
  logger.info({ viewName, durationMs }, 'Materialized view refreshed');

  // Redisキャッシュも無効化
  await redis.del(`cache:view:${viewName}:*`);

  return { durationMs };
}

// src/db/materializedViews/scheduler.ts — BullMQスケジューラー

import { Queue, Worker, QueueScheduler } from 'bullmq';

const refreshQueue = new Queue('mv-refresh', { connection: redis });

// 5分ごとに全ビューをリフレッシュ
export async function scheduleRefreshJobs(): Promise<void> {
  // 既存の繰り返しジョブを削除して再登録
  await refreshQueue.removeRepeatableByKey('mv-refresh-daily-revenue');
  await refreshQueue.removeRepeatableByKey('mv-refresh-cohort');

  await refreshQueue.add(
    'refresh-daily-revenue',
    { viewName: 'mv_daily_revenue' },
    { repeat: { every: 5 * 60 * 1000 }, jobId: 'mv-refresh-daily-revenue' }
  );

  await refreshQueue.add(
    'refresh-user-cohort',
    { viewName: 'mv_user_cohort' },
    { repeat: { every: 60 * 60 * 1000 }, jobId: 'mv-refresh-cohort' } // 1時間ごと
  );

  logger.info('Materialized view refresh jobs scheduled');
}

const worker = new Worker('mv-refresh', async (job) => {
  const { viewName } = job.data;
  const result = await refreshView(viewName);
  return result;
}, { connection: redis, concurrency: 1 }); // 同時実行1（DBへの負荷制御）
```

```typescript
// src/db/materializedViews/queries.ts — ビューからのクエリ

export interface DailyRevenueRow {
  saleDate: Date;
  category: string;
  orderCount: number;
  uniqueBuyers: number;
  revenue: number;
  avgOrderValue: number;
}

export async function getDailyRevenue(
  tenantId: string,
  fromDate: Date,
  toDate: Date,
  category?: string
): Promise<DailyRevenueRow[]> {
  const cacheKey = `cache:view:mv_daily_revenue:${tenantId}:${fromDate.toISOString()}:${toDate.toISOString()}:${category ?? 'all'}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // マテリアライズドビューから直接SELECT（通常のテーブルと同じ）
  const rows = await prisma.$queryRaw<DailyRevenueRow[]>`
    SELECT
      sale_date AS "saleDate",
      category,
      order_count AS "orderCount",
      unique_buyers AS "uniqueBuyers",
      revenue,
      avg_order_value AS "avgOrderValue"
    FROM mv_daily_revenue
    WHERE tenant_id = ${tenantId}
      AND sale_date BETWEEN ${fromDate} AND ${toDate}
      ${category ? prisma.$queryRaw`AND category = ${category}` : prisma.$queryRaw``}
    ORDER BY sale_date DESC
  `;

  // 5分キャッシュ（ビューの更新頻度に合わせる）
  await redis.set(cacheKey, JSON.stringify(rows), { EX: 300 });
  return rows;
}

// 手動リフレッシュAPIエンドポイント（管理者用）
router.post('/admin/materialized-views/:viewName/refresh', requireAdmin, async (req, res) => {
  const { viewName } = req.params;

  if (!MATERIALIZED_VIEWS.some(v => v.name === viewName)) {
    return res.status(400).json({ error: `Unknown view: ${viewName}` });
  }

  const result = await refreshView(viewName);
  res.json({ success: true, ...result });
});
```

---

## まとめ

Claude CodeでPostgreSQLマテリアライズドビューを設計する：

1. **CLAUDE.md** に適用基準（JOIN+GROUP BY+集計）・REFRESH CONCURRENTLY（本番必須）・5分更新間隔を明記
2. **CONCURRENT REFRESH** はUNIQUEインデックスが前提——ビューにもインデックスを張ることで更新後のSELECTも高速
3. **BullMQスケジューラー** で優先度別の更新間隔——売上は5分・コホートは1時間など頻度を使い分けてDB負荷を分散
4. **Redisキャッシュ+ビュー更新で二重高速化** ——ビュー更新と同時にキャッシュも無効化して次アクセス時にフレッシュなデータを5分キャッシュ

---

*パフォーマンス設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
