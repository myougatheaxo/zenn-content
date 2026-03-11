---
title: "Claude Codeでデータパイプラインを設計する：ETL・ストリーム処理・データ品質チェック"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "analytics"]
published: true
---

## はじめに

「毎日夜間に外部データを取り込みたい」「ユーザーイベントをリアルタイムで集計したい」——データパイプラインの設計にはバッチ・ストリームの選択と、データ品質の確保が必要だ。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにデータパイプライン設計ルールを書く

```markdown
## データパイプライン設計ルール

### バッチ vs ストリーム
- バッチ: 夜間ETL、レポート生成（遅延許容・大量データ）
- ストリーム: リアルタイム集計、アラート（低遅延必須）
- Lambda Architecture: バッチとストリームを組み合わせる

### データ品質
- スキーマ検証（Zodまたはdbt）でデータ品質を保証
- null・重複・異常値のチェック
- 品質スコアをメタデータに記録（下流で活用）

### エラーハンドリング
- パイプラインは冪等（再実行しても同じ結果）
- 失敗したレコードはDLQに保存（スキップして続行）
- 処理結果のチェックポイントを保存（再実行時に続きから）
```

---

## データパイプラインの生成

```
注文データのETLパイプラインを設計してください。

ソース: 外部ECサービスのAPI（1日1回）
変換: 通貨換算・カテゴリマッピング・重複排除
宛先: PostgreSQL（分析用DB）
要件：
- 冪等実行
- データ品質チェック
- 失敗レコードのDLQ
- 進捗チェックポイント

生成ファイル: src/pipeline/
```

---

## 生成されるETLパイプライン

```typescript
// src/pipeline/orderEtl.ts
import { z } from 'zod';

// ソースデータのスキーマ（外部API）
const SourceOrderSchema = z.object({
  id: z.string(),
  external_order_id: z.string(),
  amount: z.number().positive(),
  currency: z.enum(['USD', 'EUR', 'JPY', 'GBP']),
  category_code: z.string(),
  created_at: z.string().datetime(),
  customer_email: z.string().email().nullable(),
});

// 変換後のスキーマ（分析用DB）
const TransformedOrderSchema = z.object({
  id: z.string(),
  externalOrderId: z.string(),
  amountJpy: z.number().int().positive(),    // 円換算
  category: z.enum(['ELECTRONICS', 'CLOTHING', 'FOOD', 'OTHER']),
  createdAt: z.date(),
  hasCustomerEmail: z.boolean(),
  etlRunId: z.string(),
  processedAt: z.date(),
});

const CATEGORY_MAP: Record<string, string> = {
  'ELEC': 'ELECTRONICS',
  'CLOTH': 'CLOTHING',
  'FOOD': 'FOOD',
};

interface PipelineOptions {
  batchSize: number;
  startDate: Date;
  endDate: Date;
}

export async function runOrderEtl(runId: string, options: PipelineOptions): Promise<PipelineResult> {
  const { batchSize, startDate, endDate } = options;

  // チェックポイント: 前回の続きから再開
  const checkpoint = await prisma.etlCheckpoint.findUnique({
    where: { runId },
  });

  let offset = checkpoint?.lastProcessedOffset ?? 0;
  let processedCount = 0;
  let failedCount = 0;
  const errors: Array<{ orderId: string; error: string }> = [];

  logger.info({ runId, startOffset: offset }, 'ETL pipeline started');

  while (true) {
    // 外部APIからバッチ取得
    const rawOrders = await fetchExternalOrders({
      startDate,
      endDate,
      offset,
      limit: batchSize,
    });

    if (rawOrders.length === 0) break; // 全件処理完了

    // バッチを並列処理
    const results = await Promise.allSettled(
      rawOrders.map(async (raw) => {
        // スキーマ検証
        const parsed = SourceOrderSchema.safeParse(raw);
        if (!parsed.success) {
          throw new DataQualityError(`Schema validation failed: ${JSON.stringify(parsed.error.errors)}`);
        }

        // 変換
        const transformed = await transformOrder(parsed.data, runId);

        // 冪等挿入（同じexternalOrderIdは1回のみ）
        await prisma.analyticsOrder.upsert({
          where: { externalOrderId: transformed.externalOrderId },
          create: transformed,
          update: {
            amountJpy: transformed.amountJpy,
            processedAt: transformed.processedAt,
            etlRunId: runId, // 再処理された場合に更新
          },
        });

        return transformed;
      })
    );

    // 成功・失敗を集計
    for (const [index, result] of results.entries()) {
      if (result.status === 'fulfilled') {
        processedCount++;
      } else {
        failedCount++;
        const rawOrder = rawOrders[index];
        errors.push({
          orderId: rawOrder.id ?? 'unknown',
          error: result.reason.message,
        });

        // 失敗レコードをDLQに保存
        await prisma.etlDlq.create({
          data: {
            runId,
            sourceRecord: rawOrder,
            error: result.reason.message,
            failedAt: new Date(),
          },
        });
      }
    }

    // チェックポイント更新（クラッシュ時の再開用）
    offset += rawOrders.length;
    await prisma.etlCheckpoint.upsert({
      where: { runId },
      create: { runId, lastProcessedOffset: offset, updatedAt: new Date() },
      update: { lastProcessedOffset: offset, updatedAt: new Date() },
    });

    logger.info({
      runId,
      offset,
      processedCount,
      failedCount,
      batchSize: rawOrders.length,
    }, 'ETL batch completed');
  }

  return { runId, processedCount, failedCount, errors };
}

// 変換処理
async function transformOrder(raw: z.infer<typeof SourceOrderSchema>, runId: string) {
  // 通貨換算
  const exchangeRate = await getExchangeRate(raw.currency, 'JPY');
  const amountJpy = Math.round(raw.amount * exchangeRate);

  // カテゴリマッピング
  const category = CATEGORY_MAP[raw.category_code] ?? 'OTHER';

  return {
    id: crypto.randomUUID(),
    externalOrderId: raw.external_order_id,
    amountJpy,
    category,
    createdAt: new Date(raw.created_at),
    hasCustomerEmail: !!raw.customer_email,
    etlRunId: runId,
    processedAt: new Date(),
  };
}
```

---

## ETL実行スケジューラー（GitHub Actions）

```yaml
# .github/workflows/etl-pipeline.yml
name: ETL Pipeline

on:
  schedule:
    - cron: '0 2 * * *'  # 毎日午前2時（JST 11時）
  workflow_dispatch:      # 手動実行も可能

jobs:
  run-etl:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci

      - name: Run ETL pipeline
        run: |
          RUN_ID="etl-$(date +%Y%m%d)"
          START_DATE="$(date -d 'yesterday' +%Y-%m-%d)"
          END_DATE="$(date +%Y-%m-%d)"

          node scripts/runEtl.js \
            --runId "$RUN_ID" \
            --startDate "$START_DATE" \
            --endDate "$END_DATE"

        env:
          DATABASE_URL: ${{ secrets.ANALYTICS_DB_URL }}
          EXTERNAL_API_KEY: ${{ secrets.EXTERNAL_API_KEY }}

      - name: Check DLQ count
        run: |
          DLQ_COUNT=$(node scripts/getDlqCount.js --runId "etl-$(date +%Y%m%d)")
          echo "DLQ count: $DLQ_COUNT"
          if [ "$DLQ_COUNT" -gt 100 ]; then
            echo "ERROR: Too many DLQ items ($DLQ_COUNT)"
            exit 1
          fi
```

---

## まとめ

Claude CodeでETLパイプラインを設計する：

1. **CLAUDE.md** にバッチ/ストリーム選択基準・冪等実行・DLQ方針を明記
2. **Zod スキーマ検証** でソースデータの品質をパイプライン入口でチェック
3. **チェックポイント** でクラッシュ時も続きから再実行可能
4. **DLQ** で失敗レコードを保存（スキップして処理を継続・後で手動修正）

---

*データパイプライン設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
