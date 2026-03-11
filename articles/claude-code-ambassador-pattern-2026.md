---
title: "Claude Codeでアンバサダーパターンを設計する：サイドカープロキシ・ネットワーク横断処理の分離・サービスメッシュへの道"
emoji: "🛸"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-20 18:00"
---

## はじめに

「リトライ・タイムアウト・mTLS・ロギングを全サービスに個別実装している」「サービスメッシュを導入する前にアウトバウンド通信を標準化したい」——アンバサダーパターン（サイドカープロキシ）でネットワーク横断処理を一箇所に集約する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにアンバサダーパターン設計ルールを書く

```markdown
## アンバサダーパターン設計ルール

### 役割
- アプリケーションに代わってアウトバウンド通信を処理
- ネットワーク横断処理（リトライ・タイムアウト・サーキットブレーカー）を集約
- アプリケーションはlocalhost経由でアンバサダーと通信

### 実装形態
- サイドカーコンテナ（本番推奨: Envoy/Nginxをサイドカーとして配置）
- インプロセスプロキシ（開発・小規模: TypeScriptで実装）
- サービスメッシュへの移行パス（Istio導入前の準備）

### アンバサダーが処理するもの
- リトライ + 指数バックオフ
- タイムアウト管理
- サーキットブレーカー
- アクセスログ・メトリクス収集
- ヘッダー付与（Correlation ID、認証トークン）
```

---

## アンバサダーパターン実装の生成

```
アンバサダーパターンを設計してください。

要件：
- インプロセスHTTPアンバサダー
- サーキットブレーカー統合
- リトライ + タイムアウト
- メトリクス収集

生成ファイル: src/infrastructure/ambassador/
```

---

## 生成されるアンバサダーパターン実装

```typescript
// src/infrastructure/ambassador/httpAmbassador.ts — HTTPアンバサダー

export interface AmbassadorConfig {
  targetUrl: string;
  timeout: number;         // リクエストタイムアウト（ms）
  retries: number;         // 最大リトライ回数
  circuitBreaker: {
    failureThreshold: number;
    successThreshold: number;
    openDuration: number;
  };
  headers?: Record<string, string>;  // 常時付与するヘッダー
}

export interface AmbassadorMetrics {
  totalRequests: number;
  successRequests: number;
  failedRequests: number;
  retriedRequests: number;
  circuitBreakerOpens: number;
  avgResponseTimeMs: number;
}

export class HttpAmbassador {
  private readonly circuitBreaker: CircuitBreaker;
  private readonly metrics: AmbassadorMetrics = {
    totalRequests: 0,
    successRequests: 0,
    failedRequests: 0,
    retriedRequests: 0,
    circuitBreakerOpens: 0,
    avgResponseTimeMs: 0,
  };
  private responseTimes: number[] = [];

  constructor(private readonly config: AmbassadorConfig) {
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: config.circuitBreaker.failureThreshold,
      successThreshold: config.circuitBreaker.successThreshold,
      openDuration: config.circuitBreaker.openDuration,
    });
  }

  // アプリケーションに代わってリクエストを実行
  async request(options: {
    method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
    path: string;
    headers?: Record<string, string>;
    body?: unknown;
    correlationId?: string;
  }): Promise<Response> {
    this.metrics.totalRequests++;
    const start = Date.now();

    try {
      const response = await this.circuitBreaker.execute(() =>
        this.requestWithRetry(options)
      );

      this.metrics.successRequests++;
      this.recordResponseTime(Date.now() - start);
      return response;
    } catch (error) {
      this.metrics.failedRequests++;
      if (error instanceof CircuitOpenError) {
        this.metrics.circuitBreakerOpens++;
      }
      throw error;
    }
  }

  private async requestWithRetry(options: {
    method: string;
    path: string;
    headers?: Record<string, string>;
    body?: unknown;
    correlationId?: string;
  }): Promise<Response> {
    const url = `${this.config.targetUrl}${options.path}`;
    const correlationId = options.correlationId ?? ulid();

    // アンバサダーが自動付与するヘッダー
    const ambassadorHeaders: Record<string, string> = {
      'X-Correlation-Id': correlationId,
      'X-Ambassador-Version': '1.0',
      ...this.config.headers,
      ...options.headers,
    };

    let lastError: Error = new Error('No attempts');

    for (let attempt = 1; attempt <= this.config.retries + 1; attempt++) {
      if (attempt > 1) {
        this.metrics.retriedRequests++;
        const delay = Math.min(1000 * Math.pow(2, attempt - 2), 10_000);
        await sleep(delay * (0.5 + Math.random() * 0.5));  // ジッター付き
      }

      try {
        const response = await fetch(url, {
          method: options.method,
          headers: {
            'Content-Type': 'application/json',
            ...ambassadorHeaders,
          },
          body: options.body ? JSON.stringify(options.body) : undefined,
          signal: AbortSignal.timeout(this.config.timeout),
        });

        // リトライ対象のステータスコード
        if ([429, 502, 503, 504].includes(response.status) && attempt <= this.config.retries) {
          const retryAfter = response.headers.get('Retry-After');
          if (retryAfter) await sleep(parseInt(retryAfter) * 1000);
          lastError = new HttpError(response.status, response.statusText);
          continue;
        }

        // アンバサダーのアクセスログ
        logger.info({
          correlationId,
          method: options.method,
          url,
          status: response.status,
          attempt,
          durationMs: Date.now() - /* start */ 0,
        }, 'Ambassador request completed');

        return response;
      } catch (error) {
        lastError = error as Error;
        if (error instanceof DOMException && error.name === 'TimeoutError') {
          logger.warn({ correlationId, url, attempt, timeout: this.config.timeout }, 'Request timeout');
        }
      }
    }

    throw lastError;
  }

  private recordResponseTime(ms: number): void {
    this.responseTimes.push(ms);
    if (this.responseTimes.length > 100) this.responseTimes.shift();
    this.metrics.avgResponseTimeMs =
      this.responseTimes.reduce((sum, t) => sum + t, 0) / this.responseTimes.length;
  }

  getMetrics(): AmbassadorMetrics { return { ...this.metrics }; }
  getCircuitState(): string { return this.circuitBreaker.state; }
}
```

```typescript
// src/infrastructure/ambassador/ambassadorRegistry.ts — アンバサダーレジストリ

export class AmbassadorRegistry {
  private readonly ambassadors = new Map<string, HttpAmbassador>();

  register(serviceId: string, config: AmbassadorConfig): HttpAmbassador {
    const ambassador = new HttpAmbassador(config);
    this.ambassadors.set(serviceId, ambassador);
    return ambassador;
  }

  get(serviceId: string): HttpAmbassador {
    const ambassador = this.ambassadors.get(serviceId);
    if (!ambassador) throw new Error(`No ambassador for service: ${serviceId}`);
    return ambassador;
  }

  // 全サービスのメトリクスを収集
  getAllMetrics(): Record<string, AmbassadorMetrics & { circuitState: string }> {
    const result: Record<string, AmbassadorMetrics & { circuitState: string }> = {};
    for (const [id, amb] of this.ambassadors) {
      result[id] = { ...amb.getMetrics(), circuitState: amb.getCircuitState() };
    }
    return result;
  }
}

// アプリケーション設定
const ambassadorRegistry = new AmbassadorRegistry();

// 各外部サービスにアンバサダーを設定
const inventoryAmbassador = ambassadorRegistry.register('inventory-service', {
  targetUrl: process.env.INVENTORY_SERVICE_URL ?? 'http://inventory:3001',
  timeout: 5_000,
  retries: 3,
  circuitBreaker: { failureThreshold: 5, successThreshold: 2, openDuration: 30_000 },
  headers: { 'X-Service-Name': 'order-service' },
});

const notificationAmbassador = ambassadorRegistry.register('notification-service', {
  targetUrl: process.env.NOTIFICATION_SERVICE_URL ?? 'http://notification:3002',
  timeout: 3_000,
  retries: 2,
  circuitBreaker: { failureThreshold: 10, successThreshold: 3, openDuration: 60_000 },
});

// ドメインサービスからの使用（URLを知らない、アンバサダー経由）
export class InventoryClient {
  constructor(private readonly ambassador: HttpAmbassador) {}

  async checkStock(productId: string): Promise<{ available: number }> {
    const response = await this.ambassador.request({
      method: 'GET',
      path: `/api/inventory/${productId}`,
    });
    if (!response.ok) throw new ServiceUnavailableError('inventory-service');
    return response.json();
  }

  async reserveStock(productId: string, quantity: number, correlationId: string): Promise<string> {
    const response = await this.ambassador.request({
      method: 'POST',
      path: `/api/inventory/reserve`,
      body: { productId, quantity },
      correlationId,  // アンバサダーが自動でヘッダーに付与
    });
    if (!response.ok) throw new InsufficientStockError(productId);
    const { reservationId } = await response.json();
    return reservationId;
  }
}

// メトリクスエンドポイント
app.get('/api/metrics/ambassadors', requireAdmin, (req, res) => {
  res.json(ambassadorRegistry.getAllMetrics());
});
```

---

## まとめ

Claude Codeでアンバサダーパターンを設計する：

1. **CLAUDE.md** にアンバサダーがリトライ・タイムアウト・サーキットブレーカーを集約・アプリケーションはアンバサダーのみと話す・サービスメッシュへの移行パスを明記
2. **`HttpAmbassador`が横断処理を一元管理** ——`fetch()`を直接呼ぶ代わりに`ambassador.request()`を呼ぶだけ。リトライ・ジッター・タイムアウト・サーキットブレーカーが自動適用される
3. **Correlation IDの自動付与** ——アンバサダーが全リクエストに`X-Correlation-Id`を付与。分散トレーシングなしでもログからリクエストの流れを追跡できる
4. **`AmbassadorRegistry`でメトリクスを一元収集** ——全外部サービスへの成功率・平均レスポンスタイム・サーキット状態を`/api/metrics/ambassadors`で確認。インフラ監視の基礎データを自動生成

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
