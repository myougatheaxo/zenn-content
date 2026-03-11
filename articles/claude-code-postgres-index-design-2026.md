---
title: "Claude CodeでPostgreSQLインデックス設計を最適化する：複合・部分・EXPLAIN ANALYZE"
emoji: "📑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "performance"]
published: true
published_at: "2026-03-14 09:00"
---

## はじめに

「インデックスを張ったのに遅い」「インデックスが多すぎてINSERTが重い」——EXPLAIN ANALYZEで実際の実行計画を読み、正しいインデックスを設計する方法をClaude Codeに生成させる。

---

## CLAUDE.mdにインデックス設計ルールを書く

```markdown
## PostgreSQLインデックス設計ルール

### インデックス設計の原則
- クエリ先行設計: まずSQLを書いてからインデックスを決める
- 複合インデックス: WHERE + ORDER BY の列を組み合わせる
- カーディナリティ: 高いもの（ID・email）から低いもの（status）の順
- 部分インデックス: WHERE status='active' など条件付き

### コスト確認
- EXPLAIN ANALYZE を毎回確認（推定でなく実際の値）
- Seq Scan がある場合はインデックス追加を検討
- Index Scan でも Rows Removed が多ければ選択性改善
- インデックス数×書き込みコスト vs クエリ速度のトレードオフ

### 自動チェック
- pg_stat_user_indexes で unused_index を定期確認
- slow query log (> 500ms) を毎日チェック
```

---

## インデックス設計の生成

```
PostgreSQLインデックス最適化戦略を設計してください。

要件：
- よくあるクエリパターン別のインデックス
- EXPLAIN ANALYZE の読み方
- 部分インデックス・複合インデックス
- 未使用インデックスの自動検出

生成ファイル: prisma/migrations/, src/db/
```

---

## 生成されるインデックス設計実装

```sql
-- ==========================================
-- パターン1: 複合インデックス
-- WHERE + ORDER BY + LIMIT の組み合わせ
-- ==========================================

-- BAD: これだと order_date のみ使われ、user_id のフィルタが遅い
CREATE INDEX idx_orders_date ON orders(order_date);

-- GOOD: user_id でフィルタ後、order_date で並び替え
-- カラム順: 等値条件(=) → 範囲条件(>, <) → ソート
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date DESC)
  WHERE deleted_at IS NULL;  -- 部分インデックス（削除済みを除外）

-- 使用クエリ:
-- SELECT * FROM orders
-- WHERE user_id = $1 AND deleted_at IS NULL
-- ORDER BY order_date DESC
-- LIMIT 20;

-- ==========================================
-- パターン2: 部分インデックス
-- 全行でなく特定条件の行だけインデックス
-- ==========================================

-- アクティブユーザーのみのインデックス（テーブルの30%がactive）
CREATE INDEX idx_users_email_active ON users(email)
  WHERE status = 'active';

-- 未処理ジョブのみ（処理済みが99%なら1%だけインデックス）
CREATE INDEX idx_jobs_pending ON jobs(created_at)
  WHERE status IN ('pending', 'processing');

-- ==========================================
-- パターン3: 関数インデックス
-- ==========================================

-- emailは小文字化して検索したい場合
CREATE INDEX idx_users_lower_email ON users(lower(email));
-- 使用: WHERE lower(email) = lower($1)

-- JSON内のフィールド検索
CREATE INDEX idx_metadata_type ON events((metadata->>'type'));

-- ==========================================
-- パターン4: カバリングインデックス（INCLUDE）
-- ==========================================

-- SELECT でよく使うカラムをINDEXに含めてIndex-Only Scanを実現
CREATE INDEX idx_orders_status_cover ON orders(status, created_at DESC)
  INCLUDE (id, user_id, total_amount)  -- SHEAPへのアクセスを回避
  WHERE status IN ('pending', 'processing');
```

```typescript
// src/db/queryAnalyzer.ts — EXPLAIN ANALYZE の自動実行

export class QueryAnalyzer {
  // クエリの実行計画を取得してレポート
  async analyze(sql: string, params: unknown[] = []): Promise<ExecutionPlan> {
    const result = await prisma.$queryRawUnsafe<ExplainResult[]>(
      `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
      ...params
    );

    const plan = result[0]['QUERY PLAN'][0];
    const root = plan.Plan;

    const report: ExecutionPlan = {
      totalTimeMs: plan['Execution Time'],
      planningTimeMs: plan['Planning Time'],
      rowsEstimated: root['Plan Rows'],
      rowsActual: root['Actual Rows'],
      hasSeqScan: this.hasNodeType(root, 'Seq Scan'),
      sharedBlocksHit: root['Shared Hit Blocks'] ?? 0,
      sharedBlocksRead: root['Shared Read Blocks'] ?? 0,
      warnings: [],
    };

    // Seq Scan検出
    if (report.hasSeqScan) {
      report.warnings.push('Sequential scan detected - consider adding index');
    }

    // 推定行数 vs 実際行数の乖離（統計が古い可能性）
    const rowEstimationError = Math.abs(report.rowsActual - report.rowsEstimated) / Math.max(report.rowsEstimated, 1);
    if (rowEstimationError > 10) {
      report.warnings.push(`Row estimation error ${(rowEstimationError * 100).toFixed(0)}% - consider ANALYZE`);
    }

    // キャッシュミス率
    const totalBlocks = report.sharedBlocksHit + report.sharedBlocksRead;
    if (totalBlocks > 0) {
      const cacheHitRate = report.sharedBlocksHit / totalBlocks;
      if (cacheHitRate < 0.9) {
        report.warnings.push(`Cache hit rate low: ${(cacheHitRate * 100).toFixed(1)}%`);
      }
    }

    return report;
  }

  private hasNodeType(plan: ExplainNode, type: string): boolean {
    if (plan['Node Type'] === type) return true;
    return (plan.Plans ?? []).some(child => this.hasNodeType(child, type));
  }
}

// src/db/indexMonitor.ts — 未使用インデックス監視

export async function findUnusedIndexes(): Promise<UnusedIndex[]> {
  const result = await prisma.$queryRaw<UnusedIndex[]>`
    SELECT
      schemaname,
      tablename,
      indexname,
      idx_scan as scans,
      pg_size_pretty(pg_relation_size(indexrelid)) as index_size
    FROM pg_stat_user_indexes
    JOIN pg_index USING (indexrelid)
    WHERE
      idx_scan < 10                    -- 10回未満しか使われていない
      AND NOT indisprimary              -- 主キー除外
      AND NOT indisunique               -- ユニーク制約除外
      AND pg_relation_size(indexrelid) > 1024 * 1024  -- 1MB超のみ
    ORDER BY pg_relation_size(indexrelid) DESC
  `;

  return result;
}

export async function findSlowQueries(thresholdMs = 500): Promise<SlowQuery[]> {
  const result = await prisma.$queryRaw<SlowQuery[]>`
    SELECT
      query,
      calls,
      mean_exec_time,
      total_exec_time,
      rows / calls as avg_rows
    FROM pg_stat_statements
    WHERE mean_exec_time > ${thresholdMs}
    ORDER BY mean_exec_time DESC
    LIMIT 20
  `;

  return result;
}
```

```typescript
// scripts/index-health-check.ts — CIでインデックスヘルスチェック

async function runIndexHealthCheck(): Promise<void> {
  const analyzer = new QueryAnalyzer();

  // 主要クエリの実行計画チェック
  const criticalQueries = [
    {
      name: 'user orders list',
      sql: 'SELECT * FROM orders WHERE user_id = $1 AND deleted_at IS NULL ORDER BY order_date DESC LIMIT 20',
      params: ['test-user-id'],
      maxTimeMs: 50,
    },
    {
      name: 'product search',
      sql: "SELECT * FROM products WHERE status = 'active' AND category = $1 ORDER BY created_at DESC LIMIT 20",
      params: ['electronics'],
      maxTimeMs: 100,
    },
  ];

  let hasIssues = false;

  for (const query of criticalQueries) {
    const plan = await analyzer.analyze(query.sql, query.params);

    if (plan.totalTimeMs > query.maxTimeMs) {
      console.error(`SLOW: ${query.name} took ${plan.totalTimeMs}ms (max: ${query.maxTimeMs}ms)`);
      hasIssues = true;
    }
    if (plan.warnings.length > 0) {
      console.warn(`WARNINGS for ${query.name}:`, plan.warnings);
      hasIssues = true;
    }
  }

  if (hasIssues) process.exit(1);
  console.log('Index health check passed');
}
```

---

## まとめ

Claude CodeでPostgreSQLインデックス設計を最適化する：

1. **CLAUDE.md** にクエリ先行設計・複合インデックスのカラム順（等値→範囲→ソート）・部分インデックスを明記
2. **INCLUDE句** でカバリングインデックスを作りTable Heapへのアクセスをゼロに——Index-Only Scan実現
3. **pg_stat_user_indexes** で10回未満使用・1MB超のインデックスを検出して不要なものを削除
4. **CIでEXPLAIN ANALYZE** を実行し、Seq Scanや推定行数乖離をデプロイ前に検出

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
