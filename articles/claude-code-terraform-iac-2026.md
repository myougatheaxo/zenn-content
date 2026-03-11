---
title: "Claude CodeでTerraform IaCを設計する：AWS ECS・RDS・VPC・State管理"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "terraform", "aws", "devops", "typescript"]
published: true
---

## はじめに

インフラをコードで管理——Terraformで AWS ECS + RDS + VPC構成を定義し、`terraform plan/apply`でゼロ手動操作のインフラ構築を実現する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにTerraform設計ルールを書く

```markdown
## Terraform IaC設計ルール

### 構造
- モジュール: vpc / ecs / rds / alb に分割
- 環境: staging / production は tfvars で分離
- Stateファイル: S3 + DynamoDBロック（絶対にローカル保存しない）

### 命名規約
- リソース: {project}-{env}-{service}（例: myapp-prod-ecs）
- タグ必須: Project, Environment, ManagedBy=terraform

### セキュリティ
- RDSはプライベートサブネット（直接接続不可）
- セキュリティグループはポートごとに最小権限
- シークレットはAWS Secrets Managerで管理（tfvarsに平文禁止）
```

---

## Terraformインフラの生成

```
AWS ECS + RDS + VPCのTerraformを設計してください。

要件：
- VPC（パブリック/プライベートサブネット）
- ECS Fargate（Node.jsアプリ）
- RDS PostgreSQL（マルチAZ）
- ALB（HTTPSリダイレクト）
- S3 State管理

生成ファイル: terraform/
```

---

## 生成されるTerraformコード

```hcl
# terraform/main.tf

terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Stateをリモートで管理（必須）
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "myapp-terraform-lock"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

```hcl
# terraform/modules/vpc/main.tf

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.project}-${var.env}-vpc" }
}

# パブリックサブネット（ALB用）
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = { Name = "${var.project}-${var.env}-public-${count.index + 1}" }
}

# プライベートサブネット（ECS/RDS用）
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = { Name = "${var.project}-${var.env}-private-${count.index + 1}" }
}

# NAT Gateway（プライベートサブネットからの外向き通信）
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}
```

```hcl
# terraform/modules/ecs/main.tf

# ECSクラスター
resource "aws_ecs_cluster" "main" {
  name = "${var.project}-${var.env}"

  setting {
    name  = "containerInsights"
    value = "enabled"  # CloudWatch Container Insights有効化
  }
}

# タスク定義
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project}-${var.env}-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.task_cpu
  memory                   = var.task_memory
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "app"
    image = "${var.ecr_repository_url}:${var.image_tag}"
    portMappings = [{
      containerPort = 3000
      protocol      = "tcp"
    }]
    environment = [
      { name = "NODE_ENV", value = var.env },
      { name = "PORT",     value = "3000"  }
    ]
    secrets = [
      {
        name      = "DATABASE_URL"
        valueFrom = aws_secretsmanager_secret.db_url.arn
      }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/${var.project}-${var.env}"
        awslogs-region        = var.aws_region
        awslogs-stream-prefix = "app"
      }
    }
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }
  }])
}

# ECSサービス（オートスケーリング付き）
resource "aws_ecs_service" "app" {
  name            = "${var.project}-${var.env}-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.min_capacity
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = var.target_group_arn
    container_name   = "app"
    container_port   = 3000
  }

  deployment_configuration {
    minimum_healthy_percent = 100
    maximum_percent         = 200
  }

  lifecycle {
    ignore_changes = [desired_count]  # HPAが管理するので無視
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${var.project}-${var.env}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value = 60.0  # CPU 60%でスケール
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    scale_in_cooldown  = 300  # 5分待ってからスケールダウン
    scale_out_cooldown = 60
  }
}
```

```hcl
# terraform/modules/rds/main.tf

resource "aws_db_instance" "main" {
  identifier = "${var.project}-${var.env}-db"

  engine         = "postgres"
  engine_version = "16.1"
  instance_class = var.instance_class  # db.t3.medium（staging）/ db.r6g.large（prod）

  allocated_storage     = 20
  max_allocated_storage = 100  # ストレージオートスケール

  db_name  = var.db_name
  username = var.db_username
  password = random_password.db.result

  # マルチAZ（本番のみ）
  multi_az               = var.env == "production"
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  # バックアップ
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  # 削除保護（本番のみ）
  deletion_protection = var.env == "production"
  skip_final_snapshot = var.env != "production"

  # パラメータグループ（pg_stat_statements有効化）
  parameter_group_name = aws_db_parameter_group.main.name

  tags = { Name = "${var.project}-${var.env}-db" }
}

# DBパスワードをSecrets Managerに保存
resource "aws_secretsmanager_secret" "db" {
  name = "${var.project}/${var.env}/db-credentials"
  recovery_window_in_days = 0
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = var.db_username
    password = random_password.db.result
    host     = aws_db_instance.main.endpoint
    dbname   = var.db_name
  })
}
```

---

## GitHub Actions CI/CDとの統合

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'terraform/**'
  push:
    branches: [main]
    paths:
      - 'terraform/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -var-file=production.tfvars -out=tfplan
        if: github.event_name == 'pull_request'

      - name: Terraform Apply
        run: terraform apply tfplan
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

---

## まとめ

Claude CodeでTerraform IaCを設計する：

1. **CLAUDE.md** にモジュール分割・S3リモートState・命名規約・シークレット禁止を明記
2. **S3 + DynamoDB** でStateをリモート管理してチーム共有（ローカル保存禁止）
3. **ECS Auto Scaling** でCPU 60%をターゲットに自動スケール（スケールイン5分待機）
4. **RDS** はプライベートサブネット + マルチAZ + 削除保護（本番環境のみ有効）

---

*Terraform設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
