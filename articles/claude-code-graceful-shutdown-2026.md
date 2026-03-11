---
title: "Claude Codeでグレースフルシャットダウンを実装する：Kubernetes対応の終了処理"
emoji: "🛑"
type: "tech"
topics: ["claudecode", "nodejs", "kubernetes", "docker", "本番運用"]
published: true
---

## はじめに

Kubernetesで`kubectl rollout restart`を実行したとき、処理中のリクエストが途中でブツ切りになっていないか？グレースフルシャットダウンを実装してNode.jsアプリを安全に終了させる方法をClaude Codeで生成する。

---

## CLAUDE.mdにシャットダウン方針を書く

```markdown
## グレースフルシャットダウン方針

### 必須実装
- SIGTERM/SIGINTを受信したら:
  1. 新規リクエストの受付を停止
  2. 処理中のリクエストが完了するまで待機（最大30秒）
  3. DB接続・外部接続を切断
  4. プロセスを終了

### タイムアウト
- シャットダウンタイムアウト: 30秒
- 30秒を超えた場合は強制終了（exit code 1）

### Kubernetes対応
- readinessProbe: /health/ready エンドポイント（シャットダウン時はfalseを返す）
- terminationGracePeriodSeconds: 60（Podの設定）
- preStop hook: sleep 5秒（ロードバランサーの切り替えを待つ）

### 外部リソースのクリーンアップ
- Prismaクライアントの切断
- Redisクライアントの切断
- BullMQワーカーの停止（処理中ジョブの完了を待つ）
- cron jobsの停止
```

---

## グレースフルシャットダウンの生成

```
Node.js + Expressアプリのグレースフルシャットダウンを実装してください。

要件：
- SIGTERM/SIGINTで起動
- 新規リクエスト停止 → 処理中リクエストの完了待機
- タイムアウト: 30秒
- クリーンアップ: Prisma + Redis + BullMQワーカーを順番に切断
- ログ: シャットダウンの各ステップを記録

保存先: src/shutdown.ts
```

---

## 生成されるシャットダウン処理

```typescript
// src/shutdown.ts
import { prisma } from './lib/prisma';
import { redis } from './lib/redis';
import { emailWorker } from './workers/emailWorker';
import { logger } from './lib/logger';

const SHUTDOWN_TIMEOUT_MS = 30_000;

export function setupGracefulShutdown(server: http.Server): void {
  let isShuttingDown = false;

  const shutdown = async (signal: string) => {
    if (isShuttingDown) return;
    isShuttingDown = true;

    logger.info({ signal }, 'Shutdown signal received');

    // 強制終了タイマー
    const forceExit = setTimeout(() => {
      logger.error('Shutdown timeout — forcing exit');
      process.exit(1);
    }, SHUTDOWN_TIMEOUT_MS);

    try {
      // 1. 新規リクエストの受付を停止
      logger.info('Closing HTTP server...');
      await new Promise<void>((resolve, reject) => {
        server.close((err) => err ? reject(err) : resolve());
      });
      logger.info('HTTP server closed');

      // 2. BullMQワーカーを停止（処理中ジョブの完了を待つ）
      logger.info('Closing BullMQ worker...');
      await emailWorker.close();
      logger.info('BullMQ worker closed');

      // 3. DB接続を切断
      logger.info('Disconnecting Prisma...');
      await prisma.$disconnect();
      logger.info('Prisma disconnected');

      // 4. Redis接続を切断
      logger.info('Closing Redis connection...');
      await redis.quit();
      logger.info('Redis disconnected');

      clearTimeout(forceExit);
      logger.info('Graceful shutdown complete');
      process.exit(0);
    } catch (err) {
      logger.error({ err }, 'Error during shutdown');
      process.exit(1);
    }
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));

  // 未処理のPromise rejectionもキャッチ
  process.on('unhandledRejection', (reason) => {
    logger.error({ reason }, 'Unhandled rejection');
  });
}
```

---

## ヘルスチェックエンドポイントの生成

```
Kubernetes対応のヘルスチェックエンドポイントを生成してください。

要件：
- GET /health/live: アプリが起動しているか（Liveness Probe）
  → 常にOK（起動している限り）
- GET /health/ready: リクエストを受け付けられるか（Readiness Probe）
  → DBとRedisに接続できていること
  → シャットダウン中はfalseを返す
- レスポンス: { status: 'ok'|'error', checks: { db: bool, redis: bool } }

保存先: src/routes/health.ts
```

```typescript
// 生成されるエンドポイント
router.get('/health/ready', async (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'error', reason: 'shutting down' });
  }

  const checks = {
    db: false,
    redis: false,
  };

  try {
    await prisma.$queryRaw`SELECT 1`;
    checks.db = true;
  } catch {}

  try {
    await redis.ping();
    checks.redis = true;
  } catch {}

  const allHealthy = Object.values(checks).every(Boolean);

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ok' : 'error',
    checks,
  });
});
```

---

## Kubernetes設定の生成

```
Kubernetesのデプロイメント設定を生成してください。

要件：
- terminationGracePeriodSeconds: 60
- preStop hook: sleep 5（ロードバランサーの切り替え待ち）
- readinessProbe: GET /health/ready（3秒毎、2回失敗でPodを外す）
- livenessProbe: GET /health/live（10秒毎、3回失敗で再起動）
```

---

## まとめ

Claude Codeでグレースフルシャットダウンを実装する：

1. **CLAUDE.md** にシャットダウン手順・タイムアウト・Kubernetes設定を明記
2. **シャットダウン処理** でリクエスト完了を待ってから外部リソースを切断
3. **Readiness Probe** でシャットダウン中のルーティングを防ぐ
4. **Kubernetes設定** でpreStop hookとterminationGracePeriodを適切に設定

---

*本番運用に必要な設定の漏れを自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
