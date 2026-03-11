---
title: "Claude Codeでヘルスエンドポイントを設計する：依存サービス確認・Liveness/Readiness・Kubernetes対応"
emoji: "❤️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "devops"]
published: true
published_at: "2026-03-18 11:00"
---

## はじめに

「ヘルスチェックが`200 OK`を返しているのにDBに繋がらない」「Kubernetesがポッドを再起動しすぎる」——適切なLiveness・Readiness・Startupプローブとヘルスエンドポイントを設計し、真に「ヘルシー」な状態だけを報告する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにヘルスエンドポイント設計ルールを書く

```markdown
## ヘルスエンドポイント設計ルール

### エンドポイント分離
- /health/live: Liveness（プロセスが生きているか）→ DBチェック不要
- /health/ready: Readiness（リクエストを処理できるか）→ 全依存サービスをチェック
- /health: 管理者向け詳細ステータス（認証必須）

### 依存サービスのチェック
- PostgreSQL: SELECT 1でクエリ確認
- Redis: PINGで確認
- 外部API: 軽量なAPIを叩く（ヘルスエンドポイントがあれば使う）
- タイムアウト: 各チェック最大2秒

### キャッシュ
- Readinessチェックの結果を5秒キャッシュ（連続チェックでDBを叩かない）
- Livenessはキャッシュしない（常に最新状態を報告）
```

---

## ヘルスエンドポイント実装の生成

```
ヘルスエンドポイントを設計してください。

要件：
- Liveness/Readiness/Startupの分離
- 依存サービスの並列チェック
- タイムアウト制御
- Kubernetes対応

生成ファイル: src/health/
```

---

## 生成されるヘルスエンドポイント実装

```typescript
// src/health/healthChecker.ts — ヘルスチェック

export type CheckStatus = 'ok' | 'degraded' | 'down';

export interface CheckResult {
  name: string;
  status: CheckStatus;
  durationMs: number;
  message?: string;
}

export interface HealthReport {
  status: CheckStatus;
  checks: CheckResult[];
  version: string;
  uptime: number;   // プロセス起動からの秒数
  timestamp: string;
}

export type HealthChecker = () => Promise<CheckResult>;

export class HealthCheckService {
  private readonly checks: Map<string, HealthChecker> = new Map();
  private readonly cache = new Map<string, { result: CheckResult; expiresAt: number }>();

  register(name: string, checker: HealthChecker, cacheTtlMs = 5_000): void {
    this.checks.set(name, async () => {
      // キャッシュチェック
      const cached = this.cache.get(name);
      if (cached && Date.now() < cached.expiresAt) return cached.result;

      const result = await checker();
      this.cache.set(name, { result, expiresAt: Date.now() + cacheTtlMs });
      return result;
    });
  }

  async runAll(): Promise<HealthReport> {
    const checkEntries = [...this.checks.entries()];

    const results = await Promise.allSettled(
      checkEntries.map(([, checker]) => checker())
    );

    const checks: CheckResult[] = results.map((r, i) => {
      if (r.status === 'fulfilled') return r.value;
      return {
        name: checkEntries[i][0],
        status: 'down' as const,
        durationMs: 0,
        message: (r.reason as Error).message,
      };
    });

    const overallStatus: CheckStatus =
      checks.some(c => c.status === 'down') ? 'down' :
      checks.some(c => c.status === 'degraded') ? 'degraded' : 'ok';

    return {
      status: overallStatus,
      checks,
      version: process.env.APP_VERSION ?? 'unknown',
      uptime: process.uptime(),
      timestamp: new Date().toISOString(),
    };
  }

  async runReadiness(): Promise<HealthReport> {
    return this.runAll();
  }

  // Livenessは軽量チェックのみ（メモリ使用量など）
  async runLiveness(): Promise<{ status: 'ok' | 'down'; memoryMb: number }> {
    const memMb = process.memoryUsage().heapUsed / 1024 / 1024;
    const memLimit = parseInt(process.env.MEMORY_LIMIT_MB ?? '512');

    return {
      status: memMb < memLimit * 0.9 ? 'ok' : 'down',
      memoryMb: Math.round(memMb),
    };
  }
}

// 各依存サービスのチェッカー
export function createPostgresChecker(db: PrismaClient): HealthChecker {
  return async () => {
    const start = Date.now();
    try {
      await Promise.race([
        db.$queryRaw`SELECT 1`,
        new Promise<never>((_, reject) => setTimeout(() => reject(new Error('Timeout')), 2000)),
      ]);
      return { name: 'postgresql', status: 'ok', durationMs: Date.now() - start };
    } catch (error) {
      return { name: 'postgresql', status: 'down', durationMs: Date.now() - start, message: (error as Error).message };
    }
  };
}

export function createRedisChecker(client: RedisClient): HealthChecker {
  return async () => {
    const start = Date.now();
    try {
      await Promise.race([
        client.ping(),
        new Promise<never>((_, reject) => setTimeout(() => reject(new Error('Timeout')), 2000)),
      ]);
      return { name: 'redis', status: 'ok', durationMs: Date.now() - start };
    } catch (error) {
      return { name: 'redis', status: 'down', durationMs: Date.now() - start, message: (error as Error).message };
    }
  };
}

export function createExternalApiChecker(name: string, url: string): HealthChecker {
  return async () => {
    const start = Date.now();
    try {
      const response = await fetch(url, { signal: AbortSignal.timeout(2000) });
      const status: CheckStatus = response.ok ? 'ok' : response.status < 500 ? 'degraded' : 'down';
      return { name, status, durationMs: Date.now() - start };
    } catch (error) {
      return { name, status: 'down', durationMs: Date.now() - start, message: (error as Error).message };
    }
  };
}
```

```typescript
// src/health/healthRouter.ts — ヘルスルーター（Kubernetes対応）

const healthService = new HealthCheckService();

// 依存サービスを登録（5秒キャッシュ）
healthService.register('postgresql', createPostgresChecker(prisma), 5_000);
healthService.register('redis', createRedisChecker(redis), 5_000);
healthService.register('payment-api', createExternalApiChecker('payment-api', process.env.PAYMENT_API_HEALTH_URL!), 10_000);

// Kubernetes Liveness Probe: プロセスが生きているかだけ
// 失敗すると: Pod再起動
router.get('/health/live', async (req, res) => {
  const result = await healthService.runLiveness();

  if (result.status === 'ok') {
    res.status(200).json(result);
  } else {
    // メモリ使用量が90%超 → 再起動させる
    res.status(503).json(result);
  }
});

// Kubernetes Readiness Probe: リクエストを受け付けられるか
// 失敗すると: Service から除外（トラフィックを送らない）
router.get('/health/ready', async (req, res) => {
  const report = await healthService.runReadiness();

  if (report.status === 'ok') {
    res.status(200).json(report);
  } else if (report.status === 'degraded') {
    // 一部の依存が不安定: 200を返してトラフィックは受け続ける
    // （全部落とすと困る場合）
    res.status(200).json(report);
  } else {
    // 必須依存が落ちている: 503でService から除外
    res.status(503).json(report);
  }
});

// 管理者向け詳細ヘルス（認証必須）
router.get('/health', requireAdmin, async (req, res) => {
  const report = await healthService.runAll();
  const statusCode = report.status === 'ok' ? 200 : report.status === 'degraded' ? 200 : 503;
  res.status(statusCode).json(report);
});

// Kubernetes設定例
/*
livenessProbe:
  httpGet:
    path: /health/live
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
*/
```

---

## まとめ

Claude Codeでヘルスエンドポイントを設計する：

1. **CLAUDE.md** にLiveness（プロセス生死のみ）/Readiness（全依存確認）を分離・各チェック2秒タイムアウト・結果を5秒キャッシュしてDBスパイク防止を明記
2. **Liveness/Readiness分離** でKubernetesの再起動判断を適切に制御——Readinessがf失敗してもPodを再起動せず(traffic-outのみ)、Livenessが失敗したら初めてPodを再起動
3. **並列チェック** でPostgreSQL+Redis+外部APIを同時確認——逐次だと6秒かかるチェックが2秒で完了。AllSettledで1つが遅くても他の結果を捨てない
4. **degraded状態** で部分的な問題を表現——Readiness=200(degraded)で「一部不安定だがトラフィックは受け付ける」と「全部落とす」の中間の対応が可能

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
