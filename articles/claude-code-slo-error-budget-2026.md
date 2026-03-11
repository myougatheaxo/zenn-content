---
title: "Claude CodeでSLO・エラーバジェットを設計する：可用性目標・バーンレートアラート・Prometheus"
emoji: "📈"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "devops", "observability"]
published: true
published_at: "2026-03-16 18:00"
---

## はじめに

「サービスが何%稼働していれば良いのかを定量化できていない」——SLO（サービスレベル目標）とエラーバジェットで「どこまで失敗を許容できるか」を数値化し、バーンレートアラートで問題を早期検出する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにSLO設計ルールを書く

```markdown
## SLO・エラーバジェット設計ルール

### SLO定義
- 可用性SLO: 99.9%（月間ダウンタイム上限: 43.8分）
- レイテンシーSLO: P95 < 500ms（95%のリクエストが500ms以内）
- エラーレートSLO: 0.1%以下（1000リクエストに1件まで）

### エラーバジェット
- 月間エラーバジェット = (1 - SLO) × 月間リクエスト数
- 例: 99.9%SLO × 月100万reqの場合: 1,000件がバジェット
- バジェット消費50%でWarning、75%でCritical、100%でIncident

### バーンレートアラート
- 1時間バーンレート > 14.4倍: 1時間でバジェット1%消費 → Critical
- 6時間バーンレート > 6倍: 6時間でバジェット5%消費 → Warning
- 72時間バーンレート > 1倍: バジェット超過ペース → Info
```

---

## SLO実装の生成

```
SLO・エラーバジェット管理システムを設計してください。

要件：
- Prometheusメトリクス収集
- エラーバジェット計算
- バーンレートアラート
- SLOダッシュボード用データ

生成ファイル: src/observability/slo/
```

---

## 生成されるSLO実装

```typescript
// src/observability/slo/sloDefinitions.ts — SLO定義

export interface SLO {
  name: string;
  description: string;
  target: number;          // 目標値（例: 0.999 = 99.9%）
  windowDays: number;      // 計算ウィンドウ（例: 30日）
  metricQuery: string;     // Prometheusクエリ
}

export const SERVICE_SLOS: SLO[] = [
  {
    name: 'api-availability',
    description: 'API全エンドポイントの可用性（5xx以外の割合）',
    target: 0.999,  // 99.9%
    windowDays: 30,
    metricQuery: `
      1 - (
        sum(rate(http_requests_total{status=~"5.."}[5m]))
        /
        sum(rate(http_requests_total[5m]))
      )
    `,
  },
  {
    name: 'api-latency-p95',
    description: 'APIのP95レイテンシー500ms以内',
    target: 0.95,  // 95%のリクエストが基準内
    windowDays: 30,
    metricQuery: `
      histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
      < 0.5
    `,
  },
];
```

```typescript
// src/observability/slo/metricsCollector.ts — Prometheusメトリクス

import { Counter, Histogram, Registry } from 'prom-client';

const registry = new Registry();

// HTTPリクエストメトリクス
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [registry],
});

const httpRequestDurationSeconds = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [registry],
});

// SLOエラーバジェット消費メトリクス
const errorBudgetConsumed = new Counter({
  name: 'slo_error_budget_consumed_total',
  help: 'Number of SLO-violating events',
  labelNames: ['slo'],
  registers: [registry],
});

// Expressメトリクスミドルウェア
export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();
  const route = req.route?.path ?? req.path;

  res.on('finish', () => {
    const durationSeconds = (Date.now() - start) / 1000;
    const labels = { method: req.method, route, status: res.statusCode.toString() };

    httpRequestsTotal.inc(labels);
    httpRequestDurationSeconds.observe(labels, durationSeconds);

    // SLOバジェット消費チェック
    if (res.statusCode >= 500) {
      errorBudgetConsumed.inc({ slo: 'api-availability' });
    }
    if (durationSeconds > 0.5) {
      errorBudgetConsumed.inc({ slo: 'api-latency-p95' });
    }
  });

  next();
}

// Prometheusスクレイピングエンドポイント
router.get('/metrics', async (req, res) => {
  res.set('Content-Type', registry.contentType);
  res.end(await registry.metrics());
});
```

```typescript
// src/observability/slo/errorBudgetCalculator.ts — エラーバジェット計算

export class ErrorBudgetCalculator {
  async calculate(sloName: string, windowDays = 30): Promise<{
    sloTarget: number;
    currentSLO: number;
    errorBudgetTotal: number;     // 月間総許容エラー数
    errorBudgetConsumed: number;  // 実際に消費したエラー数
    errorBudgetRemaining: number;
    burnRate: number;             // 現在の消費速度（1.0 = 予算通りペース）
    estimatedDaysUntilExhausted?: number;
  }> {
    // Prometheus APIから取得（実装省略、概念を示す）
    const totalRequests = await this.queryPrometheus(
      `sum(increase(http_requests_total[${windowDays}d]))`
    );

    const errorRequests = await this.queryPrometheus(
      `sum(increase(http_requests_total{status=~"5.."}[${windowDays}d]))`
    );

    const slo = SERVICE_SLOS.find(s => s.name === sloName)!;
    const currentSLO = 1 - (errorRequests / totalRequests);
    const errorBudgetTotal = (1 - slo.target) * totalRequests;
    const errorBudgetConsumed = errorRequests;
    const errorBudgetRemaining = Math.max(0, errorBudgetTotal - errorBudgetConsumed);

    // バーンレート: 実際の消費速度 / 期待消費速度
    const expectedBurnRate = 1.0; // 期待: 1日1/30を消費
    const actualBurnRate = (errorBudgetConsumed / errorBudgetTotal) / (windowDays / 30);

    const daysUntilExhausted = errorBudgetRemaining > 0 && actualBurnRate > 1
      ? (errorBudgetRemaining / errorBudgetTotal) * windowDays / actualBurnRate
      : undefined;

    return {
      sloTarget: slo.target,
      currentSLO,
      errorBudgetTotal,
      errorBudgetConsumed,
      errorBudgetRemaining,
      burnRate: actualBurnRate,
      estimatedDaysUntilExhausted: daysUntilExhausted,
    };
  }
}

// バーンレートアラートルール（Prometheus AlertManager用）
export const ALERTMANAGER_RULES = `
groups:
  - name: slo-burn-rate
    rules:
      # 1時間バーンレート > 14.4: Critical（バジェットが1時間で1%消費）
      - alert: SLOBurnRateCritical
        expr: |
          (
            sum(rate(slo_error_budget_consumed_total{slo="api-availability"}[1h]))
            /
            (sum(rate(http_requests_total[1h])) * 0.001)
          ) > 14.4
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "SLO burn rate critical: {{ $labels.slo }}"
          description: "Current burn rate: {{ $value | humanize }}x"

      # 6時間バーンレート > 6: Warning
      - alert: SLOBurnRateWarning
        expr: |
          (
            sum(rate(slo_error_budget_consumed_total{slo="api-availability"}[6h]))
            /
            (sum(rate(http_requests_total[6h])) * 0.001)
          ) > 6
        for: 15m
        labels:
          severity: warning
`;

// SLOダッシュボードAPI
router.get('/api/admin/slo/:name', requireAdmin, async (req, res) => {
  const calculator = new ErrorBudgetCalculator();
  const budget = await calculator.calculate(req.params.name);

  const status =
    budget.errorBudgetConsumed / budget.errorBudgetTotal >= 1.0 ? 'exhausted' :
    budget.errorBudgetConsumed / budget.errorBudgetTotal >= 0.75 ? 'critical' :
    budget.errorBudgetConsumed / budget.errorBudgetTotal >= 0.50 ? 'warning' : 'ok';

  res.json({ ...budget, status });
});
```

---

## まとめ

Claude CodeでSLO・エラーバジェットを設計する：

1. **CLAUDE.md** に99.9%可用性SLO・P95レイテンシー500ms・月間エラーバジェット計算式・バーンレートアラート閾値（14.4倍/6倍）を明記
2. **Prometheusメトリクス** でhttp_requests_total・レイテンシーヒストグラムを収集——resのfinishイベントでSLO違反を検出してバジェット消費カウントを増加
3. **バーンレート計算** で「今の速度で行くと何日でバジェットが尽きるか」を予測——予算超過前に対処できる
4. **AlertManagerルール** でバーンレート超過を自動アラート——Critical（1h × 14.4倍）とWarning（6h × 6倍）の二段階で異なる緊急度を表現

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
