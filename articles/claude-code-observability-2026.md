---
title: "Claude Codeで可観測性を設計する：OpenTelemetryで分散トレーシングを導入する"
emoji: "🔭"
type: "tech"
topics: ["claudecode", "opentelemetry", "nodejs", "typescript", "observability"]
published: true
---

## はじめに

本番で「なぜこのAPIが遅いのか」「どのサービスでエラーが起きているのか」が分からない——可観測性の3本柱（ログ・メトリクス・トレース）をClaude Codeに設計させる。

---

## CLAUDE.mdに可観測性ルールを書く

```markdown
## 可観測性ルール

### 3本柱（全て必須）
1. ログ: pino（構造化JSON、PIIマスキング）
2. メトリクス: Prometheus形式（prom-client）
3. トレース: OpenTelemetry（分散トレーシング）

### トレース設定
- 全HTTPリクエストにトレースIDを付与
- 外部API呼び出しとDB操作はSpanを作成
- エラー発生時はSpanにエラー情報を記録
- サンプリングレート: 本番10%、開発100%

### メトリクス
- リクエスト数: http_requests_total（method, route, statusCode）
- レイテンシ: http_request_duration_ms（histogram）
- エラーレート: http_errors_total（statusCode, route）
- カスタムビジネスメトリクス: 注文数・売上・ユーザー登録数

### アラート基準
- エラーレート > 1% → 警告
- p99レイテンシ > 1秒 → 警告
- p99レイテンシ > 3秒 → 緊急
```

---

## OpenTelemetryの設定生成

```
OpenTelemetryを使った分散トレーシングを設定してください。

要件：
- 自動インストルメンテーション: HTTP・Express・Prisma・Redis
- エクスポーター: OTLP（Jaeger or Grafana Tempo）
- リソース属性: service.name, service.version, deployment.environment
- サンプリング: TraceIdRatioBased（本番10%, 開発100%）
- TypeScript

生成するファイル:
- src/instrumentation.ts（初期化、main.tsより前にimport）
```

```typescript
// src/instrumentation.ts（アプリの最初にimport）
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { TraceIdRatioBased } from '@opentelemetry/sdk-trace-base';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  resource: new Resource({
    'service.name': process.env.SERVICE_NAME ?? 'my-service',
    'service.version': process.env.npm_package_version ?? '0.0.0',
    'deployment.environment': process.env.NODE_ENV ?? 'development',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318/v1/traces',
  }),
  sampler: new TraceIdRatioBased(
    process.env.NODE_ENV === 'production' ? 0.1 : 1.0
  ),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false }, // ファイルIOは除外
    }),
  ],
});

sdk.start();

process.on('SIGTERM', () => sdk.shutdown());
```

---

## Prometheusメトリクスの生成

```
prom-clientを使ったPrometheusメトリクスを設定してください。

メトリクス:
- http_requests_total: Counter（method, route, statusCode）
- http_request_duration_ms: Histogram（method, route）
- active_connections: Gauge（現在の接続数）
- db_query_duration_ms: Histogram（operation, table）

エンドポイント: GET /metrics（Prometheus形式でスクレイプ）

生成ファイル:
- src/lib/metrics.ts（メトリクス定義）
- src/middleware/metricsMiddleware.ts（リクエストメトリクス自動収集）
```

```typescript
// src/lib/metrics.ts
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

export const register = new Registry();
collectDefaultMetrics({ register }); // Node.jsのデフォルトメトリクス

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_ms',
  help: 'HTTP request duration in milliseconds',
  labelNames: ['method', 'route'],
  buckets: [10, 25, 50, 100, 250, 500, 1000, 2500, 5000],
  registers: [register],
});

// src/middleware/metricsMiddleware.ts
export function metricsMiddleware(): RequestHandler {
  return (req, res, next) => {
    const start = Date.now();

    res.on('finish', () => {
      const duration = Date.now() - start;
      const route = req.route?.path ?? req.path;

      httpRequestsTotal.labels(req.method, route, String(res.statusCode)).inc();
      httpRequestDuration.labels(req.method, route).observe(duration);
    });

    next();
  };
}
```

---

## カスタムSpanの生成

```
以下のビジネスロジックにカスタムトレースSpanを追加してください。

対象: 注文処理フロー（在庫確認 → 決済 → 注文作成）

各Spanに含める情報:
- span名: {操作名}
- 属性: orderId, userId, 金額
- エラー時: exception記録 + span.setStatus(ERROR)
```

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('order-service');

export async function processOrder(userId: string, items: OrderItem[]) {
  return tracer.startActiveSpan('order.process', async (span) => {
    span.setAttributes({ userId, itemCount: items.length });

    try {
      // 在庫確認
      await tracer.startActiveSpan('inventory.check', async (inventorySpan) => {
        inventorySpan.setAttributes({ itemIds: items.map(i => i.id).join(',') });
        await inventoryService.check(items);
        inventorySpan.end();
      });

      // 決済処理
      const payment = await tracer.startActiveSpan('payment.process', async (paySpan) => {
        const result = await paymentService.charge(userId, total);
        paySpan.setAttributes({ paymentId: result.id, amount: total });
        paySpan.end();
        return result;
      });

      span.setAttributes({ paymentId: payment.id, orderId: order.id });
      span.end();
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      span.end();
      throw err;
    }
  });
}
```

---

## まとめ

Claude Codeで可観測性を設計する：

1. **CLAUDE.md** に3本柱（ログ・メトリクス・トレース）を明記
2. **OpenTelemetry** で分散トレーシングを自動設定
3. **Prometheusメトリクス** でリクエスト数・レイテンシを自動収集
4. **カスタムSpan** でビジネスロジックの実行を可視化

---

*可観測性設定の漏れを自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
