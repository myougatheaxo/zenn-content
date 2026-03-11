---
title: "Claude Codeでクラウドコスト最適化を設計する：スポットインスタンス・S3コスト・Lambda最適化"
emoji: "💰"
type: "tech"
topics: ["claudecode", "aws", "devops", "typescript", "cost"]
published: true
---

## はじめに

AWSコストが月$5,000——スポットインスタンス・S3ライフサイクル・Lambda最適化でコスト50%削減を実現する。Claude Codeにコスト最適化設計を生成させる。

---

## CLAUDE.mdにコスト最適化設計ルールを書く

```markdown
## クラウドコスト最適化ルール

### ECS/EC2
- 本番: スポットインスタンス混在（On-Demand 30% + Spot 70%）
- 開発環境: 夜間・週末自動停止（18時〜9時、土日）
- 開発DBは夜間にスナップショット→削除（翌朝復元）

### S3コスト
- アップロード30日後: Standard-IA（アクセス頻度低）
- 90日後: Glacier Instant Retrieval
- 365日後: Glacier Deep Archive（または削除）
- 不完全マルチパートアップロードは7日で削除

### Lambda/API
- メモリは実測から最適化（Power Tuning）
- Cold start削減: Provisioned Concurrency（トラフィック予測可能なもの）
- ナイーブなポーリングは廃止→SQSトリガーに変更
```

---

## コスト最適化システムの生成

```
AWSクラウドコスト最適化を設計してください。

要件：
- ECSスポットインスタンス設定
- S3ライフサイクルポリシー
- 開発環境の自動停止・起動
- コスト異常検知

生成ファイル: terraform/cost-optimization/
```

---

## 生成されるコスト最適化実装

```hcl
# terraform/cost-optimization/ecs-spot.tf
# ECS Fargateスポット混在設定（70%コスト削減）

resource "aws_ecs_service" "app" {
  name    = "${var.project}-${var.env}-app"
  cluster = aws_ecs_cluster.main.id

  capacity_provider_strategy {
    # スポット70% + オンデマンド30%
    capacity_provider = "FARGATE_SPOT"
    weight            = 70
    base              = 0
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 30
    base              = 1  # 常に最低1タスクはオンデマンド
  }

  # スポット中断時のグレースフルシャットダウン
  deployment_configuration {
    minimum_healthy_percent = 50  # スポット中断時も50%稼働
    maximum_percent         = 200
  }
}
```

```hcl
# terraform/cost-optimization/s3-lifecycle.tf
# S3ライフサイクルポリシー

resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    id     = "uploads-lifecycle"
    status = "Enabled"

    # Standardから30日後にIA移行（アクセス頻度低いファイル）
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    # 90日後にGlacier（稀にアクセスが必要なもの）
    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }

    # 1年後に削除（法的要件がなければ）
    expiration {
      days = 365
    }
  }

  # 不完全なマルチパートアップロードを7日で削除
  rule {
    id     = "abort-multipart"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  # 古いバージョンを30日で削除（バージョニング有効時）
  rule {
    id     = "noncurrent-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}
```

```typescript
// src/cost/devEnvironmentScheduler.ts
// 開発環境の自動停止・起動（月~金 9時-18時のみ稼働）

import { ECSClient, UpdateServiceCommand, DescribeServicesCommand } from '@aws-sdk/client-ecs';
import { RDSClient, StopDBInstanceCommand, StartDBInstanceCommand } from '@aws-sdk/client-rds';

const ecs = new ECSClient({ region: 'ap-northeast-1' });
const rds = new RDSClient({ region: 'ap-northeast-1' });

// 開発環境を停止（夜間・週末）
export async function stopDevEnvironment(): Promise<void> {
  const cluster = process.env.DEV_ECS_CLUSTER!;
  const services = process.env.DEV_ECS_SERVICES!.split(',');

  // ECSサービスを0タスクにスケール
  await Promise.all(services.map(service =>
    ecs.send(new UpdateServiceCommand({
      cluster,
      service,
      desiredCount: 0,
    }))
  ));

  // RDS停止（スナップショット自動作成）
  await rds.send(new StopDBInstanceCommand({
    DBInstanceIdentifier: process.env.DEV_RDS_IDENTIFIER!,
  }));

  logger.info('Development environment stopped. Saving ~$50/day.');
}

// 開発環境を起動（平日朝）
export async function startDevEnvironment(): Promise<void> {
  const cluster = process.env.DEV_ECS_CLUSTER!;
  const services = process.env.DEV_ECS_SERVICES!.split(',');

  // RDS起動（停止から起動に約2分）
  await rds.send(new StartDBInstanceCommand({
    DBInstanceIdentifier: process.env.DEV_RDS_IDENTIFIER!,
  }));

  // RDSが利用可能になるまで待機
  let ready = false;
  for (let i = 0; i < 30; i++) {
    await new Promise(r => setTimeout(r, 10_000));
    const status = await getRdsStatus(process.env.DEV_RDS_IDENTIFIER!);
    if (status === 'available') { ready = true; break; }
  }

  if (!ready) throw new Error('RDS did not start in time');

  // ECSサービスを元のタスク数に戻す
  await Promise.all(services.map(service =>
    ecs.send(new UpdateServiceCommand({
      cluster,
      service,
      desiredCount: 2, // 開発環境は2タスク
    }))
  ));
}
```

```typescript
// src/cost/costAnomalyDetector.ts
// AWS Cost Anomaly Detectionのアラート処理

import { CostExplorerClient, GetCostAndUsageCommand } from '@aws-sdk/client-cost-explorer';

const costExplorer = new CostExplorerClient({ region: 'us-east-1' }); // Cost Explorerはus-east-1固定

// 日次コストを取得して前日比を計算
export async function checkDailyCostAnomaly(): Promise<void> {
  const today = new Date();
  const yesterday = new Date(today.getTime() - 86400 * 1000);
  const dayBefore = new Date(today.getTime() - 86400 * 2000);

  const [todayCost, yesterdayCost] = await Promise.all([
    getDailyCost(yesterday, today),
    getDailyCost(dayBefore, yesterday),
  ]);

  const changePercent = ((todayCost - yesterdayCost) / yesterdayCost) * 100;

  if (changePercent > 50) {
    // コストが50%以上増加 → Slackアラート
    await sendSlackAlert({
      text: `⚠️ コスト異常検知: 前日比 +${changePercent.toFixed(0)}%`,
      fields: [
        { title: '昨日のコスト', value: `$${yesterdayCost.toFixed(2)}` },
        { title: '今日のコスト', value: `$${todayCost.toFixed(2)}` },
        { title: '増加額', value: `$${(todayCost - yesterdayCost).toFixed(2)}` },
      ],
    });
  }
}

async function getDailyCost(start: Date, end: Date): Promise<number> {
  const result = await costExplorer.send(new GetCostAndUsageCommand({
    TimePeriod: {
      Start: start.toISOString().split('T')[0],
      End: end.toISOString().split('T')[0],
    },
    Granularity: 'DAILY',
    Metrics: ['BlendedCost'],
  }));

  return parseFloat(result.ResultsByTime?.[0]?.Total?.BlendedCost?.Amount ?? '0');
}
```

---

## まとめ

Claude Codeでクラウドコスト最適化を設計する：

1. **CLAUDE.md** にスポット70%・S3ライフサイクル・開発環境自動停止を明記
2. **ECS FARGATE_SPOT** をweight 70で混在→スポットコスト約70%削減
3. **S3ライフサイクル** で30日→IA→90日→Glacier→365日→削除と自動階層化
4. **開発環境の夜間自動停止** で1日$50節約（月$1,500削減）

---

*コスト最適化設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
