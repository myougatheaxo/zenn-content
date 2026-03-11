---
title: "Claude CodeでゼロダウンタイムDBマイグレーションを設計する：Expand-Contract・後方互換"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "devops"]
published: true
published_at: "2026-03-15 14:00"
---

## はじめに

「マイグレーション中にアプリを停止させたくない」——Expand-Contractパターンで古いコードと新しいコードを同時に動かしながら安全にスキーマ変更する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにゼロダウンタイムマイグレーション設計ルールを書く

```markdown
## ゼロダウンタイムDBマイグレーション設計ルール

### Expand-Contractパターン（4フェーズ）
1. Expand: 新しいカラム/テーブルを追加（古いものも残す）
2. Migrate: バックグラウンドでデータを新フォーマットにコピー
3. Cutover: アプリを新しいカラムで動かす（古いものも読む）
4. Contract: 古いカラムを削除

### 禁止操作（ダウンタイムなし原則）
- NOT NULLカラムの即時追加（デフォルト値ありでもロック）
- カラムのリネーム（アプリが古い名前を参照中）
- カラムの型変更（CASTが重い）

### インデックスの安全な追加
- CREATE INDEX CONCURRENTLY（テーブルロックなし）
- CONCURRENTLY中はテーブルDROPできない
- マイグレーション中エラー時はINVALID状態に注意
```

---

## ゼロダウンタイムマイグレーションの生成

```
ゼロダウンタイムDBマイグレーション戦略を設計してください。

要件：
- Expand-Contractパターン
- バックグラウンドデータ移行
- CREATE INDEX CONCURRENTLY
- マイグレーション状態管理

生成ファイル: prisma/migrations/, src/db/migrations/
```

---

## 生成されるゼロダウンタイムマイグレーション実装

```typescript
// 例: usersテーブルの full_name カラムを
//     first_name + last_name に分割するマイグレーション

// ===========================
// フェーズ1: EXPAND
// ===========================
// new migration: 20260315_001_expand_name_fields.ts
export const expand = {
  up: async (db: Kysely<any>) => {
    // 新しいカラムをNULL許可で追加（NOT NULLにしない = テーブルロックなし）
    await db.schema
      .alterTable('users')
      .addColumn('first_name', 'varchar(100)')  // NULL許可
      .addColumn('last_name', 'varchar(100)')   // NULL許可
      .execute();

    // インデックスもCONCURRENTLYで追加
    await sql`
      CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_first_name
      ON users(first_name)
      WHERE first_name IS NOT NULL
    `.execute(db);
  },
  down: async (db: Kysely<any>) => {
    await db.schema.alterTable('users')
      .dropColumn('first_name')
      .dropColumn('last_name')
      .execute();
  },
};
```

```typescript
// フェーズ2: BACKFILL（バックグラウンドデータ移行）
// 一度に全行を更新すると重いので、バッチ処理で少しずつ

export async function backfillNameFields(): Promise<void> {
  const BATCH_SIZE = 1000;
  let lastId: string | null = null;
  let totalProcessed = 0;

  logger.info('Starting name field backfill...');

  while (true) {
    // バッチ取得（first_nameがNULLの行のみ）
    const rows = await db
      .selectFrom('users')
      .select(['id', 'full_name'])
      .where('first_name', 'is', null)
      .where('full_name', 'is not', null)
      .$if(lastId !== null, qb => qb.where('id', '>', lastId!))
      .orderBy('id')
      .limit(BATCH_SIZE)
      .execute();

    if (rows.length === 0) break;

    // バッチ更新（1行ずつではなくバルク更新）
    await db.transaction().execute(async (tx) => {
      for (const row of rows) {
        const [firstName, ...rest] = (row.full_name ?? '').split(' ');
        const lastName = rest.join(' ') || null;

        await tx.updateTable('users')
          .set({ first_name: firstName || null, last_name: lastName })
          .where('id', '=', row.id)
          .execute();
      }
    });

    totalProcessed += rows.length;
    lastId = rows[rows.length - 1].id;

    logger.debug({ totalProcessed, lastId }, 'Backfill progress');

    // DBへの負荷を抑えるために少し待機
    await sleep(50);
  }

  logger.info({ totalProcessed }, 'Name field backfill completed');
}
```

```typescript
// フェーズ3: CUTOVER（アプリを新カラムで動かす）
// 両方のカラムに書き込み + 読み取りは新カラム優先

export class UserRepository {
  async createUser(input: CreateUserInput): Promise<User> {
    const [firstName, ...rest] = input.name.split(' ');
    const lastName = rest.join(' ') || null;

    return db.insertInto('users')
      .values({
        // 新カラムに書く
        first_name: firstName,
        last_name: lastName,
        // 古いカラムも引き続き書く（ロールバック時の安全弁）
        full_name: input.name,
      })
      .returningAll()
      .executeTakeFirstOrThrow();
  }

  async getUser(id: string): Promise<User> {
    const row = await db
      .selectFrom('users')
      .selectAll()
      .where('id', '=', id)
      .executeTakeFirstOrThrow();

    return {
      ...row,
      // 新カラム優先、なければ古いカラムから分割
      firstName: row.first_name ?? row.full_name?.split(' ')[0] ?? '',
      lastName: row.last_name ?? row.full_name?.split(' ').slice(1).join(' ') ?? '',
    };
  }
}
```

```sql
-- フェーズ4: CONTRACT（古いカラムを削除）
-- 全インスタンスが新コードで動いていることを確認してから実行

-- new migration: 20260325_004_contract_remove_full_name.ts

-- まず NOT NULL 制約を安全に追加（デフォルト値なしでも全行埋まっている前提）
ALTER TABLE users
  ALTER COLUMN first_name SET NOT NULL,
  ALTER COLUMN last_name SET NOT NULL;

-- 古いカラムを削除
ALTER TABLE users
  DROP COLUMN full_name;

-- COMMENT ON TABLE users IS 'Migrated from full_name to first_name + last_name on 2026-03-25';
```

```typescript
// src/db/migrations/migrationRunner.ts — マイグレーション安全実行スクリプト

export async function runMigrationWithSafety(migrationName: string): Promise<void> {
  // 1. デプロイ前チェック（古いコードがまだ動いていないか）
  const activeInstances = await getActiveInstances();
  const oldVersionInstances = activeInstances.filter(i => i.version < MINIMUM_COMPATIBLE_VERSION);

  if (oldVersionInstances.length > 0) {
    throw new Error(
      `Cannot run CONTRACT migration: ${oldVersionInstances.length} instances running old code`
    );
  }

  // 2. バックアップ確認
  const lastBackup = await getLastBackupTime();
  const backupAge = Date.now() - lastBackup.getTime();
  if (backupAge > 6 * 60 * 60 * 1000) { // 6時間以上古い
    logger.warn('Backup is older than 6 hours. Proceeding with caution...');
  }

  // 3. マイグレーション実行
  logger.info({ migrationName }, 'Starting migration...');
  const startTime = Date.now();

  await prisma.$executeRawUnsafe(
    `COMMENT ON TABLE _migration_lock IS 'Migration ${migrationName} started at ${new Date().toISOString()}'`
  );

  // 実際のマイグレーション（Prisma Migrate等）
  // execSync('npx prisma migrate deploy');

  logger.info({ migrationName, durationMs: Date.now() - startTime }, 'Migration completed');
}
```

---

## まとめ

Claude CodeでゼロダウンタイムDBマイグレーションを設計する：

1. **CLAUDE.md** にExpand-Contract 4フェーズ・NOT NULL即時追加禁止・CREATE INDEX CONCURRENTLY必須を明記
2. **バックフィル** はLAST ID追跡のカーソルバッチ処理で一括ではなく段階的にデータ移行——DBへの負荷を分散
3. **Cutoverフェーズ** で新旧両カラムに書き込みを継続——ロールバック時も古いカラムが有効なまま
4. **Contractフェーズ** は全インスタンスが新コードで動いていることを確認してから実行——バージョン確認を自動化

---

*マイグレーション設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
