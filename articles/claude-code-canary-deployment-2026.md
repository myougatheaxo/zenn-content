---
title: "Claude Codeでカナリアデプロイを設計する：段階的ロールアウト・自動ロールバック"
emoji: "🐤"
type: "tech"
topics: ["claudecode", "devops", "aws", "typescript", "github-actions"]
published: true
---

## はじめに

全ユーザーに一気にデプロイすると、バグが全員に影響する。カナリアデプロイで1%→10%→100%と段階的に展開し、問題があれば自動ロールバック。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにカナリアデプロイルールを書く

```markdown
## カナリアデプロイ設計ルール

### 段階的ロールアウト
- フェーズ1: 1%（社内ユーザー・ベータ）
- フェーズ2: 10%（早期アダプター）
- フェーズ3: 50%（一般ユーザー）
- フェーズ4: 100%（全ユーザー）
- 各フェーズ間: 最低30分待機（エラー率監視）

### 自動ロールバックトリガー
- エラー率 > 1%（5分間の移動平均）
- P99レイテンシ > 2000ms
- 5xxレスポンス率 > 0.5%
- ヘルスチェック失敗

### フラグ管理
- Redisでカナリア割合を管理（動的変更可能）
- ユーザーIDでハッシュ割り当て（一貫性）
- 特定ユーザー/テナントを強制的にカナリアに含める機能
```

---

## カナリア制御システムの生成

```
カナリアデプロイ制御システムを設計してください。

要件：
- Redisでカナリア割合を管理
- ユーザーIDベースで一貫した割り当て
- エラー率監視と自動ロールバック
- GitHub Actions連携

生成ファイル: src/canary/canaryController.ts
```

---

## 生成されるカナリア制御実装

```typescript
// src/canary/canaryController.ts
import crypto from 'crypto';

const CANARY_KEY = 'canary:percentage';
const CANARY_VERSION_KEY = 'canary:version';
const ERROR_RATE_KEY = 'canary:error_rate';

interface CanaryConfig {
  percentage: number; // 0-100
  version: string;
  enabledAt: Date;
}

export class CanaryController {
  async getConfig(): Promise<CanaryConfig> {
    const [percentage, version, enabledAt] = await Promise.all([
      redis.get(CANARY_KEY),
      redis.get(CANARY_VERSION_KEY),
      redis.get('canary:enabled_at'),
    ]);

    return {
      percentage: percentage ? parseInt(percentage) : 0,
      version: version ?? 'stable',
      enabledAt: enabledAt ? new Date(enabledAt) : new Date(),
    };
  }

  // ユーザーがカナリアバージョンを使用すべきか判定
  async shouldUseCanary(userId: string): Promise<boolean> {
    const config = await this.getConfig();
    if (config.percentage === 0) return false;
    if (config.percentage === 100) return true;

    // SHA256ハッシュで決定論的割り当て
    const hash = crypto
      .createHash('sha256')
      .update(`${userId}:canary`)
      .digest('hex');

    const bucket = parseInt(hash.slice(0, 8), 16) % 100;
    return bucket < config.percentage;
  }

  // カナリア割合を更新（GitHub Actionsから呼び出す）
  async setPercentage(percentage: number, version: string): Promise<void> {
    await redis.mSet({
      [CANARY_KEY]: percentage.toString(),
      [CANARY_VERSION_KEY]: version,
      'canary:enabled_at': new Date().toISOString(),
    });

    logger.info({ percentage, version }, 'Canary percentage updated');
  }

  // エラー率を記録
  async recordRequest(isCanary: boolean, isError: boolean): Promise<void> {
    const key = isCanary ? 'canary:metrics:canary' : 'canary:metrics:stable';
    const pipe = redis.multi();

    pipe.incr(`${key}:total`);
    if (isError) pipe.incr(`${key}:errors`);
    pipe.expire(`${key}:total`, 300); // 5分間のウィンドウ
    pipe.expire(`${key}:errors`, 300);

    await pipe.exec();
  }

  // エラー率を計算して自動ロールバック判定
  async checkAndRollback(): Promise<boolean> {
    const [canaryTotal, canaryErrors] = await Promise.all([
      redis.get('canary:metrics:canary:total'),
      redis.get('canary:metrics:canary:errors'),
    ]);

    const total = parseInt(canaryTotal ?? '0');
    const errors = parseInt(canaryErrors ?? '0');

    if (total < 100) return false; // サンプル不足

    const errorRate = errors / total;

    if (errorRate > 0.01) { // 1%超過でロールバック
      logger.error({ errorRate, total, errors }, 'Canary rollback triggered');

      await this.setPercentage(0, 'stable');

      // アラート送信
      await sendAlert(`🚨 Canary rollback triggered: error rate ${(errorRate * 100).toFixed(2)}%`);

      return true; // ロールバック実行
    }

    return false;
  }
}

export const canary = new CanaryController();
```

---

## ミドルウェアで自動ルーティング

```typescript
// src/middleware/canaryRouting.ts
export function canaryMiddleware() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.user?.id;

    if (!userId) {
      res.locals.isCanary = false;
      return next();
    }

    const isCanary = await canary.shouldUseCanary(userId);
    res.locals.isCanary = isCanary;

    // カナリアバージョンへのリクエストヘッダー付与
    if (isCanary) {
      req.headers['x-canary-version'] = 'true';
    }

    // レスポンス後にメトリクス記録
    res.on('finish', async () => {
      const isError = res.statusCode >= 500;
      await canary.recordRequest(isCanary, isError);

      // エラー率チェック（非同期、レスポンスをブロックしない）
      if (isCanary && isError) {
        canary.checkAndRollback().catch(err =>
          logger.error({ err }, 'Rollback check failed')
        );
      }
    });

    next();
  };
}
```

---

## GitHub Actionsでのカナリアデプロイ

```yaml
# .github/workflows/canary-deploy.yml
name: Canary Deploy

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: true
      target_percentage:
        description: 'Target canary percentage (1/10/50/100)'
        required: true
        default: '1'

jobs:
  canary-deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Deploy canary (ECS weighted routing)
        run: |
          # ECSサービスをカナリアタスク定義で更新
          aws ecs update-service \
            --cluster production \
            --service myapp-canary \
            --task-definition myapp:${{ inputs.image_tag }} \
            --desired-count 2  # 全体の10%相当

      - name: Update canary percentage via API
        run: |
          curl -X POST ${{ secrets.API_URL }}/internal/canary \
            -H "Authorization: Bearer ${{ secrets.INTERNAL_TOKEN }}" \
            -d '{"percentage": ${{ inputs.target_percentage }}, "version": "${{ inputs.image_tag }}"}'

      - name: Monitor error rate (5 minutes)
        run: |
          for i in {1..10}; do
            sleep 30
            RATE=$(curl -s ${{ secrets.API_URL }}/internal/canary/metrics | jq '.canary.errorRate')
            echo "Error rate: ${RATE}"

            if (( $(echo "$RATE > 0.01" | bc -l) )); then
              echo "ERROR RATE TOO HIGH - Triggering rollback"
              curl -X POST ${{ secrets.API_URL }}/internal/canary/rollback \
                -H "Authorization: Bearer ${{ secrets.INTERNAL_TOKEN }}"
              exit 1
            fi
          done

      - name: Promote to next stage
        if: ${{ inputs.target_percentage != '100' }}
        run: |
          echo "✅ Canary healthy at ${{ inputs.target_percentage }}%"
          echo "Run this workflow again with target_percentage=100 to complete rollout"
```

---

## まとめ

Claude Codeでカナリアデプロイを設計する：

1. **CLAUDE.md** に段階的割合・自動ロールバックトリガー・監視期間を明記
2. **SHA256ハッシュ** でユーザーIDから一貫したカナリア割り当て
3. **5分間ウィンドウ** でエラー率監視（過去の累積ではなく直近を見る）
4. **GitHub Actions** + `workflow_dispatch` で段階的な割合更新を制御

---

*カナリアデプロイ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
