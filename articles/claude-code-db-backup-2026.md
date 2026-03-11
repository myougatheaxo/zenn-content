---
title: "Claude CodeでDBバックアップ・復旧を設計する：自動バックアップ・PITR・障害復旧テスト"
emoji: "💾"
type: "tech"
topics: ["claudecode", "postgresql", "aws", "devops", "typescript"]
published: true
published_at: "2026-03-12 12:00"
---

## はじめに

バックアップは取っているが復旧をテストしたことがない——そのバックアップは使い物にならない可能性がある。自動バックアップ・PITRとClaude Codeで定期的な復旧テスト自動化を設計する。

---

## CLAUDE.mdにDBバックアップ設計ルールを書く

```markdown
## DBバックアップ・復旧設計ルール

### バックアップ戦略
- 完全バックアップ: 毎日深夜2時
- WALアーカイブ: 継続的（PITR対応）
- 保持期間: 本番30日、ステージング7日
- 暗号化: AES-256（S3 SSE-S3）

### RTO/RPO目標
- RTO（復旧時間）: 4時間以内
- RPO（データ損失許容）: 1時間以内
- PITR（ポイントインタイムリカバリ）: 5分単位

### 復旧テスト
- 月次: バックアップから本番同等環境に復元
- 四半期: 完全DRドリル（本番→ステージングへの切り替え）
- テスト後: データ整合性チェック（行数・チェックサム比較）
```

---

## DBバックアップシステムの生成

```
PostgreSQLバックアップ・復旧システムを設計してください。

要件：
- pg_dumpによる定期バックアップ
- WALアーカイブ（PITR対応）
- S3暗号化保存
- 月次復旧テスト自動化

生成ファイル: src/backup/
```

---

## 生成されるバックアップ実装

```typescript
// src/backup/backupService.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { exec } from 'child_process';
import { promisify } from 'util';
import { createReadStream, createWriteStream } from 'fs';
import { createGzip } from 'zlib';
import { pipeline } from 'stream/promises';

const execAsync = promisify(exec);
const s3 = new S3Client({ region: 'ap-northeast-1' });

// PostgreSQL完全バックアップ
export async function performFullBackup(): Promise<BackupResult> {
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  const backupFile = `/tmp/backup-${timestamp}.dump`;
  const gzipFile = `${backupFile}.gz`;

  try {
    // pg_dump（カスタム形式: 並列復元対応）
    await execAsync(
      `pg_dump --format=custom --compress=0 --no-password \
       --dbname="${process.env.DATABASE_URL}" \
       --file="${backupFile}"`,
      { env: { ...process.env, PGPASSWORD: process.env.DB_PASSWORD } }
    );

    // gzip圧縮（通常60-70%圧縮）
    await pipeline(
      createReadStream(backupFile),
      createGzip({ level: 6 }),
      createWriteStream(gzipFile)
    );

    // S3にアップロード（サーバーサイド暗号化）
    const s3Key = `backups/full/${new Date().toISOString().split('T')[0]}/${timestamp}.dump.gz`;
    await s3.send(new PutObjectCommand({
      Bucket: process.env.BACKUP_S3_BUCKET!,
      Key: s3Key,
      Body: createReadStream(gzipFile),
      ServerSideEncryption: 'aws:kms',
      SSEKMSKeyId: process.env.BACKUP_KMS_KEY_ID!,
      Metadata: {
        'db-name': process.env.DB_NAME!,
        'backup-type': 'full',
        'pg-version': await getPgVersion(),
        'row-count-estimate': String(await getRowCountEstimate()),
      },
    }));

    // バックアップ記録をDBに保存
    const record = await prisma.backupRecord.create({
      data: {
        s3Key,
        type: 'full',
        sizeBytes: await getFileSize(gzipFile),
        checksum: await computeChecksum(gzipFile),
        status: 'completed',
        completedAt: new Date(),
      },
    });

    logger.info({ s3Key, record }, 'Full backup completed');
    return { s3Key, recordId: record.id };
  } finally {
    // 一時ファイルを削除
    await execAsync(`rm -f "${backupFile}" "${gzipFile}"`).catch(() => {});
  }
}

// ポイントインタイムリカバリ（PITR）設定
export async function configurePITR(): Promise<void> {
  // PostgreSQL WALアーカイブ設定（postgresql.confへの追記）
  const walConfig = `
# WAL Archive Settings for PITR
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://${process.env.BACKUP_S3_BUCKET}/wal-archive/%f'
archive_timeout = 300  # 5分ごとに強制アーカイブ（RPO: 5分）
  `;
  // 設定適用はDBA作業のため、設定内容をログ出力
  logger.info({ walConfig }, 'WAL archive config (apply manually)');
}
```

```typescript
// src/backup/recoveryTest.ts — 月次復旧テスト自動化

// 最新バックアップをテスト環境に復元して整合性確認
export async function performRecoveryTest(): Promise<RecoveryTestResult> {
  const testDb = `recovery_test_${Date.now()}`;
  let passed = false;

  try {
    // 最新バックアップを取得
    const latestBackup = await prisma.backupRecord.findFirst({
      where: { type: 'full', status: 'completed' },
      orderBy: { completedAt: 'desc' },
    });

    if (!latestBackup) throw new Error('No backup found');

    // S3からバックアップをダウンロード
    const localFile = `/tmp/${testDb}.dump.gz`;
    await execAsync(`aws s3 cp s3://${process.env.BACKUP_S3_BUCKET}/${latestBackup.s3Key} ${localFile}`);

    // テスト用DBを作成して復元
    await execAsync(`createdb ${testDb}`);
    await execAsync(
      `gunzip -c ${localFile} | pg_restore --no-owner --no-privileges --dbname=${testDb}`
    );

    // 整合性チェック
    const checks = await runIntegrityChecks(testDb);
    passed = checks.every(c => c.passed);

    // 復旧時間を記録
    await prisma.backupRecord.update({
      where: { id: latestBackup.id },
      data: {
        lastTestedAt: new Date(),
        lastTestPassed: passed,
        recoveryTimeMinutes: checks[0]?.durationMinutes,
      },
    });

    if (!passed) {
      await sendAlert('backup_test_failed', { checks });
    }

    return { passed, checks, backupAge: latestBackup.completedAt };
  } finally {
    await execAsync(`dropdb --if-exists ${testDb}`).catch(() => {});
    await execAsync(`rm -f /tmp/${testDb}.dump.gz`).catch(() => {});
  }
}

async function runIntegrityChecks(dbName: string): Promise<IntegrityCheck[]> {
  const checks: IntegrityCheck[] = [];

  // テーブル行数の比較（本番 vs 復元後）
  const tables = ['users', 'orders', 'products'];
  for (const table of tables) {
    const [prodCount, testCount] = await Promise.all([
      getTableRowCount('production', table),
      getTableRowCount(dbName, table),
    ]);

    checks.push({
      name: `${table}_row_count`,
      passed: testCount >= prodCount * 0.99, // 1%以内の差異は許容
      details: { prodCount, testCount },
      durationMinutes: 0,
    });
  }

  // 外部キー制約チェック
  const fkViolations = await execAsync(
    `psql -d ${dbName} -c "SELECT COUNT(*) FROM orders o LEFT JOIN users u ON o.user_id = u.id WHERE u.id IS NULL" -t`
  );
  checks.push({
    name: 'foreign_key_constraints',
    passed: parseInt(fkViolations.stdout.trim()) === 0,
    details: { violations: parseInt(fkViolations.stdout.trim()) },
    durationMinutes: 0,
  });

  return checks;
}
```

---

## GitHub Actions: 月次復旧テスト自動化

```yaml
# .github/workflows/recovery-test.yml
name: Monthly Recovery Test

on:
  schedule:
    - cron: '0 3 1 * *'  # 毎月1日の午前3時
  workflow_dispatch:       # 手動実行も可能

jobs:
  recovery-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: Run recovery test
        run: |
          npx ts-node -e "
          import { performRecoveryTest } from './src/backup/recoveryTest';
          performRecoveryTest().then(result => {
            if (!result.passed) {
              console.error('Recovery test FAILED:', result);
              process.exit(1);
            }
            console.log('Recovery test PASSED:', result);
          });
          "

      - name: Notify result
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "Monthly Recovery Test: ${{ job.status == 'success' && '✅ PASSED' || '❌ FAILED' }}"}
```

---

## まとめ

Claude CodeでDBバックアップ・復旧を設計する：

1. **CLAUDE.md** に完全バックアップ毎日・WALアーカイブ5分・RPO1時間・月次復旧テストを明記
2. **pg_dumpカスタム形式** で並列復元対応（大規模DBも高速復旧）
3. **チェックサムとS3メタデータ** でバックアップの完整性を記録
4. **月次自動復旧テスト** でバックアップの実際の使用可能性を定期検証

---

*DBバックアップ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
