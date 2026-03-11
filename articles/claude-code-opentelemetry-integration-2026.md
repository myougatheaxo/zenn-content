---
title: "OpenTelemetry統合実践：分散トレーシング・メトリクス・ログの計装"
emoji: "🔭"
type: "tech"
topics:
  - claudecode
  - observability
  - opentelemetry
  - typescript
published: true
---

## OpenTelemetryとは何か：なぜ今必要なのか

OpenTelemetry（OTel）は、分散システムの可観測性（Observability）を実現するオープンスタンダードだ。トレース・メトリクス・ログの3つの柱を統一されたAPIで扱える。

マイクロサービスやAI Agentシステムでは、「どのリクエストがどこで遅延しているか」「どのLLM呼び出しがコストを食っているか」を把握するのが困難だ。OpenTelemetryはその問題を解決する。

バックエンドは自由に選択できる:
- **Jaeger**: OSSの分散トレーシング（セルフホスト）
- **Grafana Tempo + Prometheus**: メトリクス+トレースの統合
- **Datadog / Honeycomb**: マネージドサービス
- **Signoz**: OSSのフルスタック可観測性

## 基本セットアップ：Node.jsへの計装

```typescript
// src/instrumentation.ts - アプリ起動前に必ず実行
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { Resource } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";

const resource = new Resource({
  [SemanticResourceAttributes.SERVICE_NAME]: "my-ai-service",
  [SemanticResourceAttributes.SERVICE_VERSION]: process.env.npm_package_version ?? "0.0.0",
  [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV ?? "development",
});

const traceExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? "http://localhost:4318/v1/traces",
});

const metricExporter = new OTLPMetricExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? "http://localhost:4318/v1/metrics",
});

export const sdk = new NodeSDK({
  resource,
  traceExporter,
  metricReader: new PeriodicExportingMetricReader({
    exporter: metricExporter,
    exportIntervalMillis: 15_000,
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      // HTTP・PostgreSQL・Redis等を自動計装
      "@opentelemetry/instrumentation-http": { enabled: true },
      "@opentelemetry/instrumentation-pg": { enabled: true },
    }),
  ],
});

sdk.start();

process.on("SIGTERM", () => {
  sdk.shutdown().finally(() => process.exit(0));
});
```

```typescript
// src/index.ts - 計装後にアプリをインポート
import "./instrumentation";  // 最初にロード
import { app } from "./app";

app.listen(3000);
```

## カスタムトレース：LLM呼び出しを計装する

```typescript
import { trace, context, SpanStatusCode, SpanKind } from "@opentelemetry/api";
import Anthropic from "@anthropic-ai/sdk";

const tracer = trace.getTracer("llm-service", "1.0.0");

async function tracedLLMCall(
  prompt: string,
  model: string = "claude-sonnet-4-5",
): Promise<string> {
  return tracer.startActiveSpan(
    "llm.call",
    {
      kind: SpanKind.CLIENT,
      attributes: {
        "llm.model": model,
        "llm.prompt_length": prompt.length,
        "llm.provider": "anthropic",
      },
    },
    async (span) => {
      try {
        const client = new Anthropic();
        const startTime = Date.now();

        const response = await client.messages.create({
          model,
          max_tokens: 1024,
          messages: [{ role: "user", content: prompt }],
        });

        const latencyMs = Date.now() - startTime;
        const result = response.content[0].text;

        // トレースにLLMメトリクスを記録
        span.setAttributes({
          "llm.input_tokens": response.usage.input_tokens,
          "llm.output_tokens": response.usage.output_tokens,
          "llm.latency_ms": latencyMs,
          "llm.stop_reason": response.stop_reason ?? "",
        });

        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error instanceof Error ? error.message : "Unknown error",
        });
        span.recordException(error as Error);
        throw error;
      } finally {
        span.end();
      }
    }
  );
}
```

## カスタムメトリクス：ビジネス指標を計測する

```typescript
import {
  metrics,
  ValueType,
} from "@opentelemetry/api";

const meter = metrics.getMeter("ai-service", "1.0.0");

// カウンター（累積値）
const requestCounter = meter.createCounter("api.requests.total", {
  description: "Total number of API requests",
  unit: "1",
});

// ヒストグラム（分布）
const latencyHistogram = meter.createHistogram("api.latency.ms", {
  description: "API request latency in milliseconds",
  unit: "ms",
  valueType: ValueType.DOUBLE,
  advice: {
    explicitBucketBoundaries: [10, 25, 50, 100, 250, 500, 1000, 2500, 5000],
  },
});

// ゲージ（現在値）
const activeConnectionsGauge = meter.createObservableGauge(
  "db.connections.active",
  {
    description: "Number of active database connections",
    unit: "connections",
  }
);
activeConnectionsGauge.addCallback((result) => {
  result.observe(pool.totalCount - pool.idleCount, { db: "primary" });
});

// アップダウンカウンター（増減する値）
const queueDepthCounter = meter.createUpDownCounter("queue.depth", {
  description: "Current queue depth",
  unit: "jobs",
});

// FastAPIルートへの統合例
async function handleRequest(req: Request): Promise<Response> {
  const startTime = Date.now();
  const labels = {
    method: req.method,
    path: new URL(req.url).pathname,
  };

  requestCounter.add(1, labels);
  queueDepthCounter.add(1);

  try {
    const response = await processRequest(req);
    latencyHistogram.record(Date.now() - startTime, {
      ...labels,
      status: String(response.status),
    });
    return response;
  } finally {
    queueDepthCounter.add(-1);
  }
}
```

## 構造化ログとトレースの相関

```typescript
import { trace } from "@opentelemetry/api";
import pino from "pino";

// ピノロガーにトレースIDを自動付与
const logger = pino({
  mixin() {
    const span = trace.getActiveSpan();
    if (!span) return {};
    const ctx = span.spanContext();
    return {
      traceId: ctx.traceId,
      spanId: ctx.spanId,
    };
  },
});

// これでログとトレースが相関付けられる
// Jaeger等でトレースIDを検索 → 対応するログが見つかる
logger.info({ userId: 123 }, "User logged in");
```

## Docker Compose：OTelコレクター + Jaegerの環境構築

```yaml
# docker-compose.observability.yml
version: "3.8"

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics

  jaeger:
    image: jaegertracing/all-in-one:1.55
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC

  prometheus:
    image: prom/prometheus:v2.50.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.3.0
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
```

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: "0.0.0.0:4317" }
      http: { endpoint: "0.0.0.0:4318" }

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls: { insecure: true }
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

OpenTelemetryの導入で「何が遅いか」「何がコストを食っているか」が可視化される。特にLLM呼び出しの計装は、AI システムの最適化に直結する重要な投資だ。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
