---
title: "Claude CodeでAPIパフォーマンスバジェットを設計する：P99目標・計測・自動アラート"
emoji: "⏱️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "performance", "observability"]
published: true
published_at: "2026-03-13 16:00"
---

## はじめに

「なんとなく遅い」ではなく数値で管理する——APIエンドポイントごとにP99目標を設定し、違反したときに自動アラートを出すパフォーマンスバジェット管理をClaude Codeに設計させる。

---

## CLAUDE.mdにパフォーマンスバジェット設計ルールを書く

```markdown
## パフォーマンスバジェット設計ルール

### レイテンシ目標（P99）
- 認証エンドポイント: 200ms
- 一覧系API: 300ms
- 詳細API: 100ms
- 書き込み系: 500ms
- 重い検索: 1000ms

### 計測と監視
- 全エンドポイントのP50/P95/P99をPrometheusに記録
- 5分間のP99がバジェット超過 → Slackアラート
- CI: 100リクエストのP95がバジェット超過 → PR失敗

### 最適化プロセス
- まず計測してからボトルネックを特定（推測禁止）
- N+1クエリを検知（> 10クエリ/リクエスト）
- 遅いクエリログ（> 100ms）は全てSentryに記録
```

---

## パフォーマンスバジェット管理の生成

```
APIパフォーマンスバジェット管理システムを設計してください。

要件：
- エンドポイントごとのレイテンシ目標
- Prometheusメトリクス記録
- バジェット違反の自動アラート
- N+1クエリ検知

生成ファイル: src/performance/
```

---

## 生成されるパフォーマンスバジェット実装

```typescript
// src/performance/budget.ts

// エンドポイントごとのP99目標（ミリ秒）
const PERFORMANCE_BUDGETS: Record<string, number> = {
  'GET /api/users/:id':           100,
  'GET /api/products':            300,
  'POST /api/orders':             500,
  'POST /api/auth/login':         200,
  'GET /api/search':             1000,
  'default':                      500, // 未設定エンドポイントのデフォルト
};

// Prometheusヒストグラム
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_ms',
  help: 'HTTP request duration in milliseconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [10, 25, 50, 100, 200, 300, 500, 1000, 2000, 5000],
});

// パフォーマンスバジェット違反カウンター
const budgetViolations = new Counter({
  name: 'http_budget_violations_total',
  help: 'Number of requests exceeding performance budget',
  labelNames: ['method', 'route'],
});
```

```typescript
// src/performance/middleware.ts

export function performanceBudgetMiddleware(req: Request, res: Response, next: NextFunction) {
  const startTime = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - startTime;
    const route = req.route?.path ?? 'unknown';
    const key = `${req.method} ${route}`;
    const budget = PERFORMANCE_BUDGETS[key] ?? PERFORMANCE_BUDGETS['default'];

    // Prometheusに記録
    httpRequestDuration.observe(
      { method: req.method, route, status_code: res.statusCode },
      duration
    );

    // バジェット違反チェック
    if (duration > budget) {
      budgetViolations.inc({ method: req.method, route });

      logger.warn({
        method: req.method,
        route,
        duration,
        budget,
        overage: duration - budget,
        requestId: req.headers['x-request-id'],
        traceId: req.traceId,
      }, `Performance budget exceeded: ${key} took ${duration}ms (budget: ${budget}ms)`);

      // 著しく遅い場合（バジェットの2倍超）はSentryに記録
      if (duration > budget * 2) {
        Sentry.captureMessage('Severe performance degradation detected', {
          level: 'warning',
          extra: { route, duration, budget, query: req.query },
        });
      }
    }
  });

  next();
}
```

```typescript
// src/performance/n1detector.ts — N+1クエリ検知

export function detectN1Queries(prisma: PrismaClient): void {
  let queryCount = 0;
  const queryLog: string[] = [];

  prisma.$use(async (params, next) => {
    const start = Date.now();
    const result = await next(params);
    const duration = Date.now() - start;

    queryCount++;
    queryLog.push(`${params.model}.${params.action} (${duration}ms)`);

    // リクエスト終了時にカウントリセット（AsyncLocalStorageで管理）
    return result;
  });

  // リクエストごとのクエリ数監視
  prisma.$on('query' as any, (e: any) => {
    if (e.duration > 100) {
      logger.warn({
        query: e.query.slice(0, 200),
        duration: e.duration,
        target: e.target,
      }, 'Slow query detected');
    }
  });
}

// リクエスト内クエリ数監視ミドルウェア
export function queryCountMiddleware(req: Request, res: Response, next: NextFunction) {
  const queryTracker = { count: 0, queries: [] as string[] };

  req.queryTracker = queryTracker;
  res.on('finish', () => {
    if (queryTracker.count > 10) {
      logger.warn({
        route: req.route?.path,
        queryCount: queryTracker.count,
        queries: queryTracker.queries.slice(0, 5), // 最初の5クエリをサンプル
      }, `Possible N+1: ${queryTracker.count} queries in single request`);
    }
  });

  next();
}
```

```typescript
// src/performance/ciPerformanceTest.ts — CIパフォーマンステスト

// package.json scripts:
// "test:perf": "ts-node src/performance/ciPerformanceTest.ts"

async function runPerformanceTest() {
  const results: Record<string, number[]> = {};

  // エンドポイントごとに100リクエスト実行
  const endpoints = [
    { method: 'GET', path: '/api/products', expectedP95: 300 },
    { method: 'POST', path: '/api/orders', expectedP95: 500, body: testOrderBody },
  ];

  for (const endpoint of endpoints) {
    results[endpoint.path] = [];

    for (let i = 0; i < 100; i++) {
      const start = Date.now();
      await fetch(`${BASE_URL}${endpoint.path}`, {
        method: endpoint.method,
        body: endpoint.body ? JSON.stringify(endpoint.body) : undefined,
        headers: testHeaders,
      });
      results[endpoint.path].push(Date.now() - start);
    }
  }

  let failed = false;

  for (const endpoint of endpoints) {
    const durations = results[endpoint.path].sort((a, b) => a - b);
    const p95 = durations[Math.floor(durations.length * 0.95)];

    console.log(`${endpoint.path}: P95=${p95}ms (budget: ${endpoint.expectedP95}ms)`);

    if (p95 > endpoint.expectedP95) {
      console.error(`❌ BUDGET EXCEEDED: ${endpoint.path} P95=${p95}ms > ${endpoint.expectedP95}ms`);
      failed = true;
    }
  }

  if (failed) process.exit(1);
  console.log('✅ All performance budgets passed');
}

runPerformanceTest();
```

---

## まとめ

Claude CodeでAPIパフォーマンスバジェットを設計する：

1. **CLAUDE.md** にエンドポイントごとのP99目標・5分超過でアラート・10クエリ超でN+1検知を明記
2. **Prometheusヒストグラム** でP50/P95/P99を継続記録し、Grafanaでダッシュボード
3. **バジェット2倍超過時** はSentryに記録して重大な劣化を見逃さない
4. **CIのパフォーマンステスト** でP95がバジェット超過したPRをブロック

---

*パフォーマンス設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
