---
title: "Claude CodeでGitHub Actions本番デプロイを設計する：ECS Fargate・手動承認・スモークテスト"
emoji: "🚢"
type: "tech"
topics: ["claudecode", "github-actions", "nodejs", "aws", "devops"]
published: true
---

## はじめに

「mainにマージしたら自動で本番にデプロイ」——ただし本番デプロイには手動承認とスモークテストが必要だ。Claude Codeにパイプライン設計を生成させる。

---

## CLAUDE.mdにCI/CDルールを書く

```markdown
## CI/CDデプロイ設計ルール

### PR CI
- lint + type check
- unit test（カバレッジ80%以上）
- Docker build確認
- セキュリティスキャン（trivy）

### ステージングデプロイ（main merge後、自動）
- Dockerビルド & ECRプッシュ
- ECS Fargate更新
- デプロイ完了を待機（ecs wait）
- ヘルスチェックでスモークテスト

### 本番デプロイ（手動承認必須）
- GitHub Environmentsで承認者を設定
- workflow_dispatchでimage_tagを指定
- デプロイ後スモークテスト
- 失敗時は手動ロールバック手順を用意
```

---

## 生成されるCI/CDパイプライン

```yaml
# .github/workflows/ci.yml — PR向けCI
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm test -- --coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
      - run: docker build --target production .
      - name: Trivy security scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'HIGH,CRITICAL'
          exit-code: '1'
```

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: Build and push image
        run: |
          ECR_URL=${{ secrets.ECR_REGISTRY }}/myapp
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker build -t ${ECR_URL}:${{ github.sha }} .
          docker push ${ECR_URL}:${{ github.sha }}
          docker tag ${ECR_URL}:${{ github.sha }} ${ECR_URL}:staging-latest
          docker push ${ECR_URL}:staging-latest

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster staging \
            --service myapp \
            --force-new-deployment

      - name: Wait for stability
        run: |
          aws ecs wait services-stable \
            --cluster staging \
            --services myapp

      - name: Smoke test
        run: |
          curl -sf https://staging.example.com/health || exit 1
          curl -sf https://staging.example.com/ready || exit 1
```

```yaml
# .github/workflows/deploy-production.yml — 本番（手動承認必須）
name: Deploy Production

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy (e.g. github.sha)'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # GitHub Environmentsで承認者を設定

    steps:
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: Deploy production
        run: |
          ECR_URL=${{ secrets.ECR_REGISTRY }}/myapp
          # 指定タグのイメージを本番用タグに付け替え
          docker pull ${ECR_URL}:${{ inputs.image_tag }}
          docker tag ${ECR_URL}:${{ inputs.image_tag }} ${ECR_URL}:production-latest
          docker push ${ECR_URL}:production-latest

          aws ecs update-service \
            --cluster production \
            --service myapp \
            --force-new-deployment

      - name: Wait and smoke test
        run: |
          aws ecs wait services-stable --cluster production --services myapp
          curl -sf https://api.example.com/health || exit 1
          curl -sf https://api.example.com/ready || exit 1
```

---

## まとめ

Claude CodeでGitHub Actionsデプロイを設計する：

1. **CLAUDE.md** にブランチ戦略・CI必須チェック・承認フローを明記
2. **PR CI** でlint/test/docker build/セキュリティスキャンを全実行
3. **GitHub Environments** で本番デプロイに手動承認者を設定
4. **ecs wait** + スモークテストでデプロイ完了を確認

---

*CI/CDパイプラインの設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
