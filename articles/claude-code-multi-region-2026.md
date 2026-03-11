---
title: "Claude Codeでマルチリージョンを設計する：グローバルDB・災害復旧・レイテンシ最適化"
emoji: "🌍"
type: "tech"
topics: ["claudecode", "aws", "devops", "typescript", "database"]
published: true
published_at: "2026-03-12 10:00"
---

## はじめに

東京のサーバーに欧米ユーザーがアクセスすると200ms以上のレイテンシが発生する。マルチリージョン配置とAurora Global DatabaseでユーザーはどこからでもP99 < 100msを実現する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにマルチリージョン設計ルールを書く

```markdown
## マルチリージョン設計ルール

### リージョン選択
- プライマリ: ap-northeast-1（東京）
- セカンダリ: us-east-1（バージニア）、eu-west-1（アイルランド）
- ユーザーは最寄りリージョンにルーティング（Route53 Latency）

### データ戦略
- Aurora Global Database: プライマリ書き込み、セカンダリ読み込み
- レプリケーション遅延: 通常 < 1秒
- 書き込みはプライマリリージョンに転送（最終整合性）

### フェイルオーバー
- RTO（目標復旧時間）: 1分以内
- RPO（目標復旧ポイント）: 最大1秒のデータ損失
- フェイルオーバーは自動または手動（重要操作は手動承認）
```

---

## マルチリージョンアーキテクチャの生成

```
AWSマルチリージョン配置を設計してください。

要件：
- Route53レイテンシールーティング
- Aurora Global Database
- CloudFront（CDN）
- リージョンフェイルオーバー
- ヘルスチェック

生成ファイル: terraform/multi-region/
```

---

## 生成されるマルチリージョン実装

```hcl
# terraform/multi-region/route53.tf

# レイテンシールーティング（ユーザーを最近傍リージョンへ）
resource "aws_route53_record" "api_tokyo" {
  zone_id = var.hosted_zone_id
  name    = "api.myapp.com"
  type    = "A"

  alias {
    name                   = aws_alb.tokyo.dns_name
    zone_id                = aws_alb.tokyo.zone_id
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = "ap-northeast-1"
  }

  set_identifier = "tokyo"
  health_check_id = aws_route53_health_check.tokyo.id
}

resource "aws_route53_record" "api_virginia" {
  zone_id = var.hosted_zone_id
  name    = "api.myapp.com"
  type    = "A"

  alias {
    name                   = aws_alb.virginia.dns_name
    zone_id                = aws_alb.virginia.zone_id
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = "us-east-1"
  }

  set_identifier = "virginia"
  health_check_id = aws_route53_health_check.virginia.id
}

# ヘルスチェック（10秒間隔、3回失敗でフェイルオーバー）
resource "aws_route53_health_check" "tokyo" {
  fqdn              = aws_alb.tokyo.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  tags = { Name = "tokyo-health-check" }
}
```

```hcl
# terraform/multi-region/aurora-global.tf

# Aurora Global Database（東京プライマリ）
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "myapp-global"
  engine                    = "aurora-postgresql"
  engine_version            = "16.1"
  database_name             = "myapp"
}

# 東京クラスター（プライマリ・書き込み可）
resource "aws_rds_cluster" "tokyo" {
  provider                  = aws.tokyo
  cluster_identifier        = "myapp-tokyo"
  engine                    = "aurora-postgresql"
  engine_version            = "16.1"
  global_cluster_identifier = aws_rds_global_cluster.main.id

  master_username = var.db_username
  master_password = random_password.db.result

  backup_retention_period = 7
  skip_final_snapshot     = false

  db_subnet_group_name   = aws_db_subnet_group.tokyo.name
  vpc_security_group_ids = [aws_security_group.rds_tokyo.id]
}

# バージニアクラスター（セカンダリ・読み込み専用）
resource "aws_rds_cluster" "virginia" {
  provider                  = aws.virginia
  cluster_identifier        = "myapp-virginia"
  engine                    = "aurora-postgresql"
  engine_version            = "16.1"
  global_cluster_identifier = aws_rds_global_cluster.main.id

  # セカンダリはmaster_username/passwordなし
  db_subnet_group_name   = aws_db_subnet_group.virginia.name
  vpc_security_group_ids = [aws_security_group.rds_virginia.id]

  depends_on = [aws_rds_cluster.tokyo]
}
```

```typescript
// src/db/multiRegionClient.ts

// リージョンに応じてDBクライアントを選択
const REGION = process.env.AWS_REGION ?? 'ap-northeast-1';
const IS_PRIMARY = REGION === 'ap-northeast-1';

// 書き込みは常にプライマリ（東京）
const primaryUrl = IS_PRIMARY
  ? process.env.DATABASE_URL!
  : process.env.DATABASE_PRIMARY_URL!; // セカンダリリージョンからプライマリへの接続

// 読み込みはローカルリージョン
const replicaUrl = process.env.DATABASE_REPLICA_URL ?? process.env.DATABASE_URL!;

export const writeDb = new PrismaClient({
  datasources: { db: { url: primaryUrl } },
});

export const readDb = new PrismaClient({
  datasources: { db: { url: replicaUrl } },
});

// セカンダリリージョンからの書き込みはプライマリに転送
export async function write<T>(fn: (db: PrismaClient) => Promise<T>): Promise<T> {
  if (IS_PRIMARY) {
    return fn(writeDb);
  }

  // セカンダリリージョンではプライマリDBに書き込み（レイテンシは高くなる）
  // 書き込みが多い場合はメッセージキューで非同期化を検討
  return fn(writeDb);
}

// 読み込みはローカルリージョンから（低レイテンシ）
export async function read<T>(fn: (db: PrismaClient) => Promise<T>): Promise<T> {
  return fn(readDb);
}
```

```typescript
// src/health/regionHealth.ts — ヘルスチェックエンドポイント

export async function healthCheck(): Promise<HealthStatus> {
  const [dbHealth, redisHealth, replicaLag] = await Promise.all([
    checkDatabase(),
    checkRedis(),
    checkReplicaLag(),
  ]);

  const region = process.env.AWS_REGION ?? 'unknown';
  const healthy = dbHealth && redisHealth && replicaLag < 10;

  return {
    status: healthy ? 'healthy' : 'unhealthy',
    region,
    components: {
      database: { healthy: dbHealth },
      redis: { healthy: redisHealth },
      replicaLag: { seconds: replicaLag, healthy: replicaLag < 10 },
    },
    timestamp: new Date().toISOString(),
  };
}

async function checkReplicaLag(): Promise<number> {
  if (process.env.AWS_REGION === 'ap-northeast-1') return 0; // プライマリは遅延なし

  try {
    const result = await readDb.$queryRaw<[{ lag: number }]>`
      SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag
    `;
    return result[0]?.lag ?? 0;
  } catch {
    return 999; // チェック失敗は異常扱い
  }
}
```

---

## まとめ

Claude Codeでマルチリージョンを設計する：

1. **CLAUDE.md** にプライマリ東京・Route53レイテンシールーティング・RTO 1分・RPO 1秒を明記
2. **Aurora Global Database** でリージョン間自動レプリケーション（通常 < 1秒）
3. **Route53ヘルスチェック** で10秒間隔、3回失敗でフェイルオーバー
4. **セカンダリリージョンの書き込みはプライマリに転送** し、読み込みはローカルで低レイテンシ

---

*マルチリージョン設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
