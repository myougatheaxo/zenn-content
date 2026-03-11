---
title: "Claude Codeでヘルスチェックを設計する：Kubernetes readiness/liveness probeと依存チェック"
emoji: "🏥"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "kubernetes", "devops"]
published: true
---

## はじめに

ヘルスチェックが`/health → 200 OK`を返すだけでは意味がない——DBに繋がっていない、Redisが落ちている状態でもOKを返す。Claude Codeに正しいヘルスチェック設計を生成させる。

---

## CLAUDE.mdにヘルスチェックルールを書く

```markdown
## ヘルスチェック設計ルール

### エンドポイント
- GET /health: 軽量チェック（ロードバランサー向け、DB接続等は確認しない）
- GET /ready: 依存関係チェック（K8s readinessProbe向け）
- GET /live: プロセス生存確認（K8s livenessProbe向け）

### readinessProbe（重要）
- DBに疎通できるか（SELECT 1）
- Redisに疎通できるか
- 外部必須サービスに疎通できるか
- 1つでも失敗したら503を返す（ロードバランサーから外す）

### レスポンス形式
{ status: "ok"|"degraded", checks: { db: "ok"|"fail", redis: "ok"|"fail" } }

### タイムアウト
- 各チェックに3秒タイムアウト
- 全体チェックは5秒以内に完了
```

---

## ヘルスチェックの生成

```
Kubernetes向けヘルスチェックAPIを設計してください。

チェック対象:
- PostgreSQL (prisma.$queryRaw SELECT 1)
- Redis (redis.ping())
- 外部API (webhook endpoint ping)

要件：
- /health: 軽量（常に200、プロセス稼働確認のみ）
- /ready: DB+Redis+外部チェック、1つ失敗で503
- /live: プロセス死活確認
- 各チェックに3秒タイムアウト

生成ファイル: src/routes/health.ts
```

---

## 生成されるヘルスチェック

```typescript
// src/routes/health.ts
import { Router } from 'express';

const router = Router();

// プロセス起動時刻
const startTime = Date.now();

interface HealthCheckResult {
  status: 'ok' | 'fail';
  responseTimeMs?: number;
  error?: string;
}

async function checkDatabase(): Promise<HealthCheckResult> {
  const start = Date.now();
  try {
    await Promise.race([
      prisma.$queryRaw`SELECT 1`,
      new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 3000)),
    ]);
    return { status: 'ok', responseTimeMs: Date.now() - start };
  } catch (err) {
    return { status: 'fail', error: String(err), responseTimeMs: Date.now() - start };
  }
}

async function checkRedis(): Promise<HealthCheckResult> {
  const start = Date.now();
  try {
    await Promise.race([
      redis.ping(),
      new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 3000)),
    ]);
    return { status: 'ok', responseTimeMs: Date.now() - start };
  } catch (err) {
    return { status: 'fail', error: String(err), responseTimeMs: Date.now() - start };
  }
}

// GET /health — 軽量チェック（ロードバランサー向け）
// DB等の依存は確認しない（高頻度でコールされるため）
router.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    uptimeSeconds: Math.floor((Date.now() - startTime) / 1000),
    timestamp: new Date().toISOString(),
  });
});

// GET /ready — readinessProbe（全依存関係の確認）
router.get('/ready', async (req, res) => {
  const [db, cache] = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
  ]);

  const checks = {
    database: db.status === 'fulfilled' ? db.value : { status: 'fail', error: String(db.reason) },
    redis: cache.status === 'fulfilled' ? cache.value : { status: 'fail', error: String(cache.reason) },
  };

  const allHealthy = Object.values(checks).every(c => c.status === 'ok');

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ok' : 'degraded',
    checks,
    timestamp: new Date().toISOString(),
  });
});

// GET /live — livenessProbe（プロセス死活確認のみ）
router.get('/live', (req, res) => {
  // イベントループがブロックされていないことを確認
  res.json({ status: 'ok' });
});

export { router as healthRouter };
```

---

## Kubernetes設定

```yaml
# kubernetes/deployment.yaml
spec:
  containers:
    - name: app
      image: myapp:latest
      ports:
        - containerPort: 3000

      # livenessProbe: プロセスが死んだらPodを再起動
      livenessProbe:
        httpGet:
          path: /live
          port: 3000
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3

      # readinessProbe: 準備できていなければトラフィックを送らない
      readinessProbe:
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 15
        periodSeconds: 5
        failureThreshold: 3
        successThreshold: 1
```

---

## まとめ

Claude Codeでヘルスチェックを設計する：

1. **CLAUDE.md** に3種のエンドポイント・チェック内容・タイムアウト値を明記
2. **/health** は軽量（ロードバランサーの高頻度コール向け）
3. **/ready** でDB+Redis+外部依存を確認、1つ失敗で503
4. **K8s設定** でlivenessProbe(/live)とreadinessProbe(/ready)を分ける

---

*ヘルスチェックの設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
