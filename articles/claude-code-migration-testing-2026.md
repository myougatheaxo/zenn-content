---
title: "Claude CodeでDBマイグレーションテストを設計する：スキーマ検証・ロールバック確認・本番前チェック"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "testing"]
published: true
published_at: "2026-03-16 20:00"
---

## はじめに

「マイグレーションを本番で実行したらエラーが出た」「ロールバックが必要になったがdownスクリプトが壊れていた」——マイグレーションをCI/CDでテストし、down/upの往復検証・本番前のdry-runを設計をClaude Codeに生成させる。

---

## CLAUDE.mdにマイグレーションテスト設計ルールを書く

```markdown
## DBマイグレーションテスト設計ルール

### CI/CDでの必須チェック
1. マイグレーション適用（up）→ スキーマ検証
2. ロールバック（down）→ 元のスキーマに戻っているか確認
3. 再適用（up）→ 冪等性チェック（2回適用しても同じ結果）
4. データマイグレーション（既存データへの影響テスト）

### 本番前dry-run
- staging環境で本番DBのスナップショットに適用
- マイグレーション実行時間を計測（長時間ロックを検出）
- 推定ロック時間: 100万行のテーブルでのALTER TABLE時間を事前測定

### 安全基準
- 実行時間 > 5秒: レビュー必須
- テーブルロック有り: ゼロダウンタイムマイグレーションへの変更を要求
- down()なし: デプロイブロック
```

---

## マイグレーションテスト実装の生成

```
DBマイグレーションテストシステムを設計してください。

要件：
- CI/CDでのup/down往復テスト
- スキーマスナップショット比較
- データマイグレーションの正確性テスト
- 本番前dry-run

生成ファイル: tests/migrations/, src/db/migrations/validator/
```

---

## 生成されるマイグレーションテスト実装

```typescript
// tests/migrations/migrationTestRunner.ts — マイグレーションテストランナー

import { execSync } from 'child_process';
import { PrismaClient } from '@prisma/client';

export class MigrationTestRunner {
  private readonly testDb: PrismaClient;
  private readonly testDbUrl: string;

  constructor() {
    // テスト用DBはCI環境のPostgreSQL
    this.testDbUrl = process.env.TEST_DATABASE_URL!;
    this.testDb = new PrismaClient({ datasources: { db: { url: this.testDbUrl } } });
  }

  // 全マイグレーションのup/down/upサイクルをテスト
  async runMigrationCycle(): Promise<MigrationTestResult[]> {
    const results: MigrationTestResult[] = [];

    // 1. クリーンな状態から開始
    await this.resetDatabase();

    // 2. 1マイグレーションずつ適用してスナップショット比較
    const migrations = await this.getMigrationFiles();

    for (const migration of migrations) {
      const result = await this.testSingleMigration(migration);
      results.push(result);

      if (!result.success) break; // 失敗したら停止
    }

    return results;
  }

  private async testSingleMigration(migrationFile: string): Promise<MigrationTestResult> {
    const startTime = Date.now();

    try {
      // Before: スキーマスナップショット取得
      const schemaBefore = await this.getSchemaSnapshot();

      // UP: マイグレーション適用
      const upStartMs = Date.now();
      execSync(`DATABASE_URL="${this.testDbUrl}" npx prisma migrate deploy --preview-feature`, {
        env: { ...process.env, DATABASE_URL: this.testDbUrl },
      });
      const upDurationMs = Date.now() - upStartMs;

      const schemaAfterUp = await this.getSchemaSnapshot();

      // スキーマが変わっていることを確認
      if (JSON.stringify(schemaBefore) === JSON.stringify(schemaAfterUp)) {
        return { migration: migrationFile, success: false, error: 'Migration had no effect on schema' };
      }

      // DOWN: ロールバック
      const downStartMs = Date.now();
      await this.runDownMigration(migrationFile);
      const downDurationMs = Date.now() - downStartMs;

      const schemaAfterDown = await this.getSchemaSnapshot();

      // ロールバック後に元のスキーマに戻っているか確認
      if (JSON.stringify(schemaBefore) !== JSON.stringify(schemaAfterDown)) {
        return {
          migration: migrationFile,
          success: false,
          error: 'Schema after rollback does not match schema before migration',
          schemaDiff: this.diffSchemas(schemaBefore, schemaAfterDown),
        };
      }

      // UP再適用（冪等性チェック）
      execSync(`DATABASE_URL="${this.testDbUrl}" npx prisma migrate deploy`, {
        env: { ...process.env, DATABASE_URL: this.testDbUrl },
      });

      return {
        migration: migrationFile,
        success: true,
        upDurationMs,
        downDurationMs,
        totalMs: Date.now() - startTime,
      };
    } catch (error) {
      return {
        migration: migrationFile,
        success: false,
        error: (error as Error).message,
        totalMs: Date.now() - startTime,
      };
    }
  }

  // PostgreSQLスキーマスナップショット
  private async getSchemaSnapshot(): Promise<SchemaSnapshot> {
    const tables = await this.testDb.$queryRaw<Array<{
      table_name: string;
      column_name: string;
      data_type: string;
      is_nullable: string;
      column_default: string | null;
    }>>`
      SELECT table_name, column_name, data_type, is_nullable, column_default
      FROM information_schema.columns
      WHERE table_schema = 'public'
      ORDER BY table_name, ordinal_position
    `;

    const indexes = await this.testDb.$queryRaw<Array<{
      indexname: string;
      tablename: string;
      indexdef: string;
    }>>`
      SELECT indexname, tablename, indexdef
      FROM pg_indexes
      WHERE schemaname = 'public'
      ORDER BY tablename, indexname
    `;

    return { tables, indexes };
  }
}
```

```typescript
// tests/migrations/dataMigrationTest.ts — データマイグレーションテスト

// バックフィルスクリプトのテスト（例: full_name → first_name + last_name）
describe('Name field backfill migration', () => {
  beforeEach(async () => {
    // テスト用データを投入
    await testDb.user.createMany({
      data: [
        { id: 'u1', fullName: 'John Doe',      email: 'john@example.com' },
        { id: 'u2', fullName: 'Jane',           email: 'jane@example.com' }, // 姓なし
        { id: 'u3', fullName: 'Bob Van der Berg', email: 'bob@example.com' }, // 複合姓
        { id: 'u4', fullName: null,             email: 'noname@example.com' }, // null
      ],
    });
  });

  it('should correctly split full_name into first_name and last_name', async () => {
    // バックフィル実行
    await backfillNameFields();

    const users = await testDb.user.findMany({ orderBy: { id: 'asc' } });

    expect(users[0]).toMatchObject({ firstName: 'John', lastName: 'Doe' });
    expect(users[1]).toMatchObject({ firstName: 'Jane', lastName: null });
    expect(users[2]).toMatchObject({ firstName: 'Bob', lastName: 'Van der Berg' });
    expect(users[3]).toMatchObject({ firstName: null, lastName: null }); // nullは保持
  });

  it('should process all records in batches', async () => {
    // 大量データのバッチ処理確認
    await testDb.user.createMany({
      data: Array.from({ length: 5000 }, (_, i) => ({
        id: `bulk-${i}`, fullName: `User ${i}`, email: `user${i}@test.com`,
      })),
    });

    await backfillNameFields();

    const unprocessed = await testDb.user.count({
      where: { firstName: null, fullName: { not: null } },
    });

    expect(unprocessed).toBe(0); // 全件処理済み
  });
});
```

```typescript
// src/db/migrations/validator/preMigrationCheck.ts — 本番前チェック

export async function runPreMigrationCheck(migrationSql: string): Promise<PreMigrationCheckResult> {
  const checks: CheckResult[] = [];

  // 1. テーブルロック分析
  const lockingOperations = [
    { pattern: /ALTER TABLE .+ ADD COLUMN .+ NOT NULL/i, risk: 'HIGH', message: 'NOT NULL ADD without DEFAULT will lock table' },
    { pattern: /ALTER TABLE .+ DROP COLUMN/i,            risk: 'MEDIUM', message: 'DROP COLUMN requires table rewrite in older PostgreSQL' },
    { pattern: /ALTER TABLE .+ ALTER COLUMN .+ TYPE/i,   risk: 'HIGH', message: 'Column type change will lock table' },
    { pattern: /CREATE INDEX(?! CONCURRENTLY)/i,          risk: 'MEDIUM', message: 'CREATE INDEX without CONCURRENTLY will lock table' },
  ];

  for (const op of lockingOperations) {
    if (op.pattern.test(migrationSql)) {
      checks.push({ type: 'lock', risk: op.risk, message: op.message });
    }
  }

  // 2. 大きなテーブルへの変更を検出（pg_class経由）
  const tableNames = extractTableNames(migrationSql);
  for (const table of tableNames) {
    const size = await getTableSize(table);
    if (size.rowCount > 1_000_000) {
      checks.push({
        type: 'large-table',
        risk: 'HIGH',
        message: `Table '${table}' has ${size.rowCount.toLocaleString()} rows. Migration may take significant time.`,
        estimatedMinutes: Math.ceil(size.rowCount / 100_000), // rough estimate
      });
    }
  }

  // 3. downスクリプトの存在確認
  if (!migrationSql.includes('-- Down:') && !migrationSql.includes('rollback')) {
    checks.push({ type: 'no-rollback', risk: 'MEDIUM', message: 'No rollback script found. Rollback will require manual intervention.' });
  }

  const hasHighRisk = checks.some(c => c.risk === 'HIGH');
  return {
    canProceed: !hasHighRisk || process.env.FORCE_MIGRATION === 'true',
    checks,
    requiresReview: hasHighRisk,
  };
}
```

---

## まとめ

Claude CodeでDBマイグレーションテストを設計する：

1. **CLAUDE.md** にCI/CDでのup/down往復テスト必須・冪等性確認・実行時間5秒超でレビュー必須・downなしはデプロイブロックを明記
2. **スキーマスナップショット比較** でrollback後に元のスキーマに戻っているかをinformation_schema経由で自動検証
3. **データマイグレーションテスト** でエッジケース（null値・複合名・大量データ）を事前に検証——「バックフィルしたら一部のデータが壊れた」を防止
4. **本番前チェック** でNOT NULL追加・型変更・インデックスなしCREATE INDEXなどのロック操作を静的解析——大テーブルへの変更は推定時間も表示

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
