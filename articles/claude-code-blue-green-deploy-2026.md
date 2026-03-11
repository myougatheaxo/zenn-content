---
title: "Claude CodeでBlue-Greenデプロイを設計する：ゼロダウンタイム・瞬時ロールバック"
emoji: "🔵"
type: "tech"
topics: ["claudecode", "devops", "aws", "typescript", "github-actions"]
published: true
---

## はじめに

サービス停止なしにデプロイし、問題があれば5秒でロールバックしたい。Blue-Greenデプロイで2つの環境を用意し、ロードバランサーで瞬時に切り替える。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにBlue-Greenデプロイルールを書く

```markdown
## Blue-Greenデプロイ設計ルール

### 環境設定
- Blue: 現在の本番環境（ライブトラフィック）
- Green: 新バージョンのスタンバイ環境
- 両環境は同一のDB/Redis/SQSを共有（Stateless）
- どちらが"live"かはALBリスナーで管理

### スイッチング手順
1. Greenに新バージョンをデプロイ
2. Greenでスモークテスト実行
3. ALBルールでトラフィックをBlue→Greenに切り替え
4. 10分間メトリクス監視
5. 問題なければBlueを新バージョンに更新（次回のGreenとして）

### ロールバック条件
- エラー率 > 1%（5分間）
- P99 > 2秒
- ヘルスチェック失敗
- ロールバックはALBルール変更のみ（5秒で完了）
```

---

## Blue-Greenデプロイシステムの生成

```
AWS ALBを使ったBlue-Greenデプロイを設計してください。

要件：
- ECS Fargateで Blue/Green 2環境
- ALBのターゲットグループ切り替え
- スモークテスト
- 自動ロールバック（CloudWatch Alarm）
- GitHub Actions統合

生成ファイル: .github/workflows/blue-green-deploy.yml
```

---

## 生成されるBlue-Greenデプロイ

```yaml
# .github/workflows/blue-green-deploy.yml
name: Blue-Green Deploy

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Docker image tag to deploy'
        required: true
      environment:
        description: 'Target environment'
        default: 'production'
        type: choice
        options: [production, staging]

jobs:
  blue-green-deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1

      - name: Determine current live color
        id: current
        run: |
          # 現在どちらの色がliveかをALBタグで判定
          LIVE_COLOR=$(aws elbv2 describe-target-groups \
            --load-balancer-arn ${{ secrets.ALB_ARN }} \
            --query "TargetGroups[?contains(TargetGroupName, 'live')].TargetGroupName" \
            --output text | grep -oE 'blue|green' | head -1)

          if [ "$LIVE_COLOR" = "blue" ]; then
            echo "live_color=blue" >> $GITHUB_OUTPUT
            echo "standby_color=green" >> $GITHUB_OUTPUT
          else
            echo "live_color=green" >> $GITHUB_OUTPUT
            echo "standby_color=blue" >> $GITHUB_OUTPUT
          fi

          echo "Current live: $LIVE_COLOR → deploying to: $([ $LIVE_COLOR = blue ] && echo green || echo blue)"

      - name: Deploy to standby environment
        run: |
          STANDBY_COLOR=${{ steps.current.outputs.standby_color }}
          ECR_URL=${{ secrets.ECR_REGISTRY }}/myapp

          # スタンバイECSサービスを新バージョンで更新
          aws ecs update-service \
            --cluster production-$STANDBY_COLOR \
            --service myapp \
            --task-definition myapp-${{ inputs.image_tag }} \
            --force-new-deployment

          # 安定まで待機（最大10分）
          aws ecs wait services-stable \
            --cluster production-$STANDBY_COLOR \
            --services myapp

          echo "✅ Deployed ${{ inputs.image_tag }} to $STANDBY_COLOR environment"

      - name: Smoke test standby environment
        run: |
          STANDBY_COLOR=${{ steps.current.outputs.standby_color }}
          STANDBY_URL=${{ secrets[format('APP_URL_{0}', toUpper(steps.current.outputs.standby_color))] }}

          # ヘルスチェック
          curl -sf "$STANDBY_URL/health" || exit 1
          curl -sf "$STANDBY_URL/ready" || exit 1

          # 基本機能テスト
          RESPONSE=$(curl -sf "$STANDBY_URL/api/v1/products" -H "Authorization: Bearer ${{ secrets.SMOKE_TEST_TOKEN }}")
          echo $RESPONSE | jq '.items | length' | xargs -I {} sh -c '[ {} -gt 0 ] || exit 1'

          echo "✅ Smoke tests passed for $STANDBY_COLOR environment"

      - name: Switch traffic to standby (Blue-Green swap)
        id: switch
        run: |
          STANDBY_COLOR=${{ steps.current.outputs.standby_color }}
          LIVE_COLOR=${{ steps.current.outputs.live_color }}

          STANDBY_TG_ARN=$(aws elbv2 describe-target-groups \
            --names "myapp-$STANDBY_COLOR" \
            --query "TargetGroups[0].TargetGroupArn" \
            --output text)

          LISTENER_ARN=$(aws elbv2 describe-listeners \
            --load-balancer-arn ${{ secrets.ALB_ARN }} \
            --query "Listeners[?Port==`443`].ListenerArn" \
            --output text)

          # ALBルールを変更してトラフィックを切り替え
          aws elbv2 modify-listener \
            --listener-arn "$LISTENER_ARN" \
            --default-actions "Type=forward,TargetGroupArn=$STANDBY_TG_ARN"

          echo "🔀 Traffic switched: $LIVE_COLOR → $STANDBY_COLOR"
          echo "switched_at=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_OUTPUT

      - name: Monitor for 10 minutes post-switch
        run: |
          echo "Monitoring error rate for 10 minutes..."

          for i in {1..20}; do
            sleep 30

            # CloudWatch メトリクスをチェック
            ERROR_RATE=$(aws cloudwatch get-metric-statistics \
              --namespace AWS/ApplicationELB \
              --metric-name HTTPCode_Target_5XX_Count \
              --dimensions Name=LoadBalancer,Value=${{ secrets.ALB_ARN_SUFFIX }} \
              --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
              --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
              --period 300 \
              --statistics Sum \
              --query "Datapoints[0].Sum" \
              --output text 2>/dev/null || echo "0")

            echo "Minute $(( i / 2 ))/10: 5xx count = $ERROR_RATE"

            if (( $(echo "$ERROR_RATE > 50" | bc -l) )); then
              echo "❌ High error rate detected! Triggering rollback..."
              exit 1
            fi
          done

          echo "✅ 10-minute monitoring passed. Deployment successful!"

      - name: Rollback on failure
        if: failure() && steps.switch.outcome == 'success'
        run: |
          LIVE_COLOR=${{ steps.current.outputs.live_color }}  # 旧liveに戻す

          LIVE_TG_ARN=$(aws elbv2 describe-target-groups \
            --names "myapp-$LIVE_COLOR" \
            --query "TargetGroups[0].TargetGroupArn" \
            --output text)

          LISTENER_ARN=$(aws elbv2 describe-listeners \
            --load-balancer-arn ${{ secrets.ALB_ARN }} \
            --query "Listeners[?Port==`443`].ListenerArn" \
            --output text)

          aws elbv2 modify-listener \
            --listener-arn "$LISTENER_ARN" \
            --default-actions "Type=forward,TargetGroupArn=$LIVE_TG_ARN"

          echo "⏪ Rolled back to $LIVE_COLOR environment"
```

---

## DBスキーマの後方互換性確保

```typescript
// Blue-Greenデプロイではカラム追加時に既存バージョンへの影響を考慮

// ❌ 危険: NOT NULL制約のある新規カラムは既存バージョンが書き込めない
// ALTER TABLE orders ADD COLUMN new_field VARCHAR(255) NOT NULL;

// ✅ 安全なパターン（3フェーズでのカラム追加）
// Phase 1: NULLableで追加（Blue-Greenを通過できる）
// ALTER TABLE orders ADD COLUMN new_field VARCHAR(255);

// Phase 2: 新バージョンが両カラムに書き込む（移行期間）

// Phase 3: NOT NULL制約を追加（全レコードにデータが入った後）
// ALTER TABLE orders ALTER COLUMN new_field SET NOT NULL;

// Prisma migrationでの実装例
// migration.sql:
// ALTER TABLE "orders" ADD COLUMN "new_field" TEXT;
// UPDATE "orders" SET "new_field" = 'default_value' WHERE "new_field" IS NULL;
// ALTER TABLE "orders" ALTER COLUMN "new_field" SET NOT NULL;
```

---

## まとめ

Claude CodeでBlue-Greenデプロイを設計する：

1. **CLAUDE.md** に環境定義・スイッチング手順・ロールバック条件を明記
2. **スタンバイ環境でスモークテスト** してからALBトラフィックを切り替え
3. **ALBルール変更** のみでロールバック（5秒で完了、ECSのデプロイ不要）
4. **DBスキーマ変更** は3フェーズで後方互換性を保つ（Blue-Greenの両バージョンが動く）

---

*Blue-Greenデプロイ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
