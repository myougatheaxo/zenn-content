---
title: "Claude CodeでOpenTelemetryを設計する：分散トレーシング・メトリクス・ログの統合"
emoji: "🔭"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "observability", "opentelemetry"]
published: true
---

## はじめに

マイクロサービスでリクエストが複数サービスをまたぐと、どこで遅くなったか分からない。OpenTelemetryで分散トレーシングを実装し、ボトルネックを可視化する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにOpenTelemetry設計ルールを書く

```markdown
## OpenTelemetry設計ルール

### 計装（Instrumentation）
- HTTP/DB/Redis/外部APIへのアクセスは必ず計装
- 自動計装（@opentelemetry/auto-instrumentations-node）を基本とする
- 重要なビジネスロジックには手動スパンを追加

### Trace伝播
- HTTPヘッダー（W3C TraceContext）でサービス間にtraceIdを伝播
- サービス境界を越えるリクエストは必ず子スパンを作成
- traceIdをログにも含める（ログとトレースを相関させる）

### メトリクス
- 標準メトリクス: request_count, request_duration, error_rate
- ビジネスメトリクス: order_created_total, payment_success_rate
- ヒストグラムはP50/P90/P99を記録

### エクスポート先
- 開発: Jaeger（ローカル）
- 本番: Datadog / Grafana Tempo / AWS X-Ray
```

---

## OpenTelemetry実装の生成

```
OpenTelemetryで分散トレーシングとメトリクスを設計してください。

要件：
- Node.js自動計装
- 手動スパン（重要な処理）
- カスタムメトリクス
- Jaeger/Datadogへのエクスポート

生成ファイル: src/telemetry/setup.ts
```

---

## 生成されるOpenTelemetry実装

```typescript
// src/telemetry/setup.ts（エントリーポイントの最初に読み込む）
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME ?? 'api-server',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION ?? '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV ?? 'development',
  }),

  // トレースエクスポーター（Jaeger/Collectorへ送信）
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318/v1/traces',
  }),

  // メトリクスエクスポーター（30秒ごと）
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318/v1/metrics',
    }),
    exportIntervalMillis: 30_000,
  }),

  // 自動計装（HTTP、Express、Prisma、Redis、fetch等）
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-redis-4': { enabled: true },
    }),
  ],
});

sdk.start();
logger.info('OpenTelemetry SDK started');

// グレースフルシャットダウン
process.on('SIGTERM', () => {
  sdk.shutdown().then(() => {
    logger.info('OpenTelemetry SDK shut down');
  });
});
```

```typescript
// src/telemetry/tracing.ts
import { trace, context, SpanStatusCode, SpanKind } from '@opentelemetry/api';

const tracer = trace.getTracer('api-server', '1.0.0');

// 手動スパン: 重要なビジネスロジックをトレース
export async function withSpan<T>(
  name: string,
  attributes: Record<string, string | number | boolean>,
  fn: () => Promise<T>
): Promise<T> {
  return tracer.startActiveSpan(name, { attributes, kind: SpanKind.INTERNAL }, async (span) => {
    try {
      const result = await fn();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (err) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: err instanceof Error ? err.message : String(err),
      });
      span.recordException(err as Error);
      throw err;
    } finally {
      span.end();
    }
  });
}

// 現在のtraceIdを取得（ログとの相関用）
export function getCurrentTraceId(): string | undefined {
  const span = trace.getActiveSpan();
  return span?.spanContext().traceId;
}
```

---

## サービスでの使用例

```typescript
// src/services/orderService.ts
export async function createOrder(input: CreateOrderInput): Promise<Order> {
  return withSpan(
    'order.create',
    {
      'order.user_id': input.userId,
      'order.item_count': input.items.length,
    },
    async () => {
      // 在庫確認（子スパンが自動生成される）
      const inventory = await withSpan(
        'inventory.check',
        { 'inventory.product_ids': input.items.map(i => i.productId).join(',') },
        () => checkInventory(input.items)
      );

      // Stripe決済（自動計装でHTTPスパンが生成される）
      const paymentIntent = await stripe.paymentIntents.create({
        amount: calculateTotal(input.items),
        currency: 'jpy',
      });

      // DB保存（Prisma自動計装でDBスパンが生成される）
      const order = await prisma.order.create({
        data: { ...input, paymentIntentId: paymentIntent.id },
      });

      return order;
    }
  );
}
```

---

## カスタムメトリクス

```typescript
// src/telemetry/metrics.ts
import { metrics } from '@opentelemetry/api';

const meter = metrics.getMeter('api-server', '1.0.0');

// カウンター
export const orderCreatedCounter = meter.createCounter('orders.created.total', {
  description: 'Total number of orders created',
});

export const paymentSuccessCounter = meter.createCounter('payments.success.total', {
  description: 'Total successful payments',
});

// ヒストグラム（レイテンシ分布）
export const bffLatencyHistogram = meter.createHistogram('bff.request.duration.ms', {
  description: 'BFF request duration in milliseconds',
  unit: 'ms',
  advice: { explicitBucketBoundaries: [10, 50, 100, 200, 500, 1000, 2000, 5000] },
});

// 使用例
export async function createOrderWithMetrics(input: CreateOrderInput) {
  const startTime = Date.now();
  try {
    const order = await createOrder(input);
    orderCreatedCounter.add(1, { 'order.status': 'success', 'order.currency': 'jpy' });
    return order;
  } catch (err) {
    orderCreatedCounter.add(1, { 'order.status': 'error' });
    throw err;
  } finally {
    bffLatencyHistogram.record(Date.now() - startTime, { endpoint: '/orders' });
  }
}
```

---

## pinoログとトレースを相関させる

```typescript
// src/lib/logger.ts
import pino from 'pino';
import { getCurrentTraceId } from './telemetry/tracing';

const baseLogger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  formatters: {
    log(obj) {
      // ログに現在のtraceIdを自動付与（Datadog等でトレースとログを相関）
      const traceId = getCurrentTraceId();
      if (traceId) {
        return { ...obj, 'trace_id': traceId };
      }
      return obj;
    },
  },
});

export const logger = baseLogger;
```

---

## まとめ

Claude CodeでOpenTelemetryを設計する：

1. **CLAUDE.md** に自動計装基本・手動スパン対象・traceIdログ付与を明記
2. **自動計装** でHTTP/DB/Redisのスパンをコードなしで取得
3. **手動スパン** で重要なビジネスロジック（注文作成・決済）のトレースを追加
4. **traceIdをログに付与** でログとトレースを相関させてデバッグを高速化

---

*OpenTelemetry設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
