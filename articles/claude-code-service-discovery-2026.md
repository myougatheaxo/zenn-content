---
title: "Claude Codeでサービスディスカバリーを設計する：動的エンドポイント解決・ヘルスチェック統合・クライアントサイドLB"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-20 13:00"
---

## はじめに

「マイクロサービスのIPアドレスがデプロイのたびに変わる」「ハードコードしたエンドポイントでダウンしたインスタンスにリクエストが流れ続ける」——サービスレジストリと動的エンドポイント解決、クライアントサイドロードバランシングを設計をClaude Codeに生成させる。

---

## CLAUDE.mdにサービスディスカバリー設計ルールを書く

```markdown
## サービスディスカバリー設計ルール

### サービスレジストリ
- 各サービスが起動時に自身のアドレス・ポート・メタデータを登録
- ヘルスチェック失敗したインスタンスは自動的に登録解除
- TTLベースのハートビートでゾンビインスタンスを除去

### エンドポイント解決
- サービス名で解決（ハードコードIPを使わない）
- ローカルキャッシュ（30秒TTL）でレジストリの負荷を軽減
- 解決失敗時のフォールバック設定

### クライアントサイドLB
- ラウンドロビン / 重み付きランダム / 最少接続数から選択
- ヘルシーなインスタンスのみにルーティング
- サーキットブレーカーと組み合わせ
```

---

## サービスディスカバリー実装の生成

```
サービスディスカバリーを設計してください。

要件：
- Redisベースのサービスレジストリ
- ハートビートによる死活管理
- クライアントサイドロードバランシング
- ローカルキャッシュ

生成ファイル: src/discovery/
```

---

## 生成されるサービスディスカバリー実装

```typescript
// src/discovery/serviceRegistry.ts — サービスレジストリ

export interface ServiceInstance {
  instanceId: string;
  serviceId: string;        // 例: 'order-service'
  address: string;          // IPまたはホスト名
  port: number;
  metadata: {
    version: string;
    zone?: string;           // 可用性ゾーン（AZ-based routing用）
    weight?: number;         // 重み付きLB用
    tags?: string[];
  };
  registeredAt: Date;
  lastHeartbeatAt: Date;
}

const REGISTRY_KEY = (serviceId: string) => `discovery:${serviceId}:instances`;
const INSTANCE_KEY = (instanceId: string) => `discovery:instance:${instanceId}`;
const HEARTBEAT_TTL = 30;  // 30秒でTTL切れ → 自動除去

export class ServiceRegistry {
  constructor(private readonly redis: Redis) {}

  // 起動時にインスタンスを登録
  async register(instance: Omit<ServiceInstance, 'registeredAt' | 'lastHeartbeatAt'>): Promise<void> {
    const now = new Date();
    const fullInstance: ServiceInstance = {
      ...instance,
      registeredAt: now,
      lastHeartbeatAt: now,
    };

    const pipeline = this.redis.pipeline();

    // インスタンス情報を保存（TTLあり）
    pipeline.setEx(
      INSTANCE_KEY(instance.instanceId),
      HEARTBEAT_TTL,
      JSON.stringify(fullInstance)
    );

    // サービスのインスタンス集合に追加
    pipeline.sAdd(REGISTRY_KEY(instance.serviceId), instance.instanceId);

    await pipeline.exec();

    logger.info({ instanceId: instance.instanceId, serviceId: instance.serviceId },
      'Service instance registered');
  }

  // ハートビート（定期実行してTTLを更新）
  async heartbeat(instanceId: string): Promise<void> {
    const raw = await this.redis.get(INSTANCE_KEY(instanceId));
    if (!raw) {
      logger.warn({ instanceId }, 'Instance not found, re-registration required');
      return;
    }

    const instance: ServiceInstance = JSON.parse(raw);
    instance.lastHeartbeatAt = new Date();

    // TTLをリセット
    await this.redis.setEx(INSTANCE_KEY(instanceId), HEARTBEAT_TTL, JSON.stringify(instance));
  }

  // シャットダウン時に登録解除
  async deregister(instanceId: string, serviceId: string): Promise<void> {
    const pipeline = this.redis.pipeline();
    pipeline.del(INSTANCE_KEY(instanceId));
    pipeline.sRem(REGISTRY_KEY(serviceId), instanceId);
    await pipeline.exec();

    logger.info({ instanceId, serviceId }, 'Service instance deregistered');
  }

  // サービスの全アクティブインスタンスを取得
  async getInstances(serviceId: string): Promise<ServiceInstance[]> {
    const instanceIds = await this.redis.sMembers(REGISTRY_KEY(serviceId));
    if (instanceIds.length === 0) return [];

    const pipeline = this.redis.pipeline();
    for (const id of instanceIds) {
      pipeline.get(INSTANCE_KEY(id));
    }
    const results = await pipeline.exec();

    const instances: ServiceInstance[] = [];
    for (let i = 0; i < results.length; i++) {
      const raw = results[i][1] as string | null;
      if (raw) {
        instances.push(JSON.parse(raw));
      } else {
        // TTL切れ（ゾンビ）→ 集合から除去
        await this.redis.sRem(REGISTRY_KEY(serviceId), instanceIds[i]);
      }
    }

    return instances;
  }
}
```

```typescript
// src/discovery/serviceDiscoveryClient.ts — クライアントサイドディスカバリー

export type LoadBalancingStrategy = 'round-robin' | 'random' | 'weighted-random' | 'least-connections';

export class ServiceDiscoveryClient {
  // ローカルキャッシュ（レジストリへのアクセスを減らす）
  private readonly cache = new Map<string, {
    instances: ServiceInstance[];
    fetchedAt: number;
  }>();
  private readonly cacheTtlMs = 30_000;  // 30秒

  // ラウンドロビン用カウンター
  private readonly rrCounters = new Map<string, number>();

  // 接続数カウンター（最少接続数LB用）
  private readonly connectionCounts = new Map<string, number>();

  constructor(
    private readonly registry: ServiceRegistry,
    private readonly strategy: LoadBalancingStrategy = 'round-robin'
  ) {}

  // サービスのエンドポイントを解決
  async resolve(serviceId: string): Promise<ServiceInstance> {
    const instances = await this.getHealthyInstances(serviceId);

    if (instances.length === 0) {
      throw new ServiceNotAvailableError(serviceId);
    }

    return this.selectInstance(serviceId, instances);
  }

  // URLを解決（ベースURL直接取得）
  async resolveUrl(serviceId: string): Promise<string> {
    const instance = await this.resolve(serviceId);
    return `http://${instance.address}:${instance.port}`;
  }

  // ヘルシーなインスタンスのみ返す（キャッシュ付き）
  private async getHealthyInstances(serviceId: string): Promise<ServiceInstance[]> {
    const cached = this.cache.get(serviceId);
    if (cached && Date.now() - cached.fetchedAt < this.cacheTtlMs) {
      return cached.instances;
    }

    // キャッシュミス → レジストリから取得
    const instances = await this.registry.getInstances(serviceId);

    // ヘルスチェック（簡易: ハートビート時刻から判定）
    const healthyInstances = instances.filter(inst => {
      const age = Date.now() - new Date(inst.lastHeartbeatAt).getTime();
      return age < HEARTBEAT_TTL * 1000 * 1.5;  // 1.5倍の猶予
    });

    this.cache.set(serviceId, { instances: healthyInstances, fetchedAt: Date.now() });
    return healthyInstances;
  }

  // ロードバランシング戦略に基づいてインスタンスを選択
  private selectInstance(serviceId: string, instances: ServiceInstance[]): ServiceInstance {
    switch (this.strategy) {
      case 'round-robin': {
        const counter = this.rrCounters.get(serviceId) ?? 0;
        const instance = instances[counter % instances.length];
        this.rrCounters.set(serviceId, counter + 1);
        return instance;
      }

      case 'random':
        return instances[Math.floor(Math.random() * instances.length)];

      case 'weighted-random': {
        // 重みの合計に対してランダム選択
        const totalWeight = instances.reduce((sum, i) => sum + (i.metadata.weight ?? 1), 0);
        let rand = Math.random() * totalWeight;
        for (const instance of instances) {
          rand -= (instance.metadata.weight ?? 1);
          if (rand <= 0) return instance;
        }
        return instances[0];
      }

      case 'least-connections': {
        // 現在の接続数が最少のインスタンスを選択
        return instances.reduce((min, inst) => {
          const minConn = this.connectionCounts.get(min.instanceId) ?? 0;
          const instConn = this.connectionCounts.get(inst.instanceId) ?? 0;
          return instConn < minConn ? inst : min;
        });
      }

      default:
        return instances[0];
    }
  }

  // 接続数のトラッキング（最少接続数LB用）
  trackConnection(instanceId: string): () => void {
    const count = this.connectionCounts.get(instanceId) ?? 0;
    this.connectionCounts.set(instanceId, count + 1);

    // 接続終了時に呼ぶクリーンアップ関数を返す
    return () => {
      const current = this.connectionCounts.get(instanceId) ?? 1;
      this.connectionCounts.set(instanceId, Math.max(0, current - 1));
    };
  }

  // キャッシュを強制更新
  invalidate(serviceId: string): void {
    this.cache.delete(serviceId);
  }
}

// ハートビートの自動管理
export class ServiceRegistrar {
  private heartbeatTimer?: NodeJS.Timeout;
  private readonly instanceId: string;

  constructor(
    private readonly registry: ServiceRegistry,
    private readonly serviceId: string
  ) {
    this.instanceId = `${serviceId}-${process.env.POD_NAME ?? ulid()}`;
  }

  async start(port: number, metadata: ServiceInstance['metadata'] = { version: '1.0.0' }): Promise<void> {
    await this.registry.register({
      instanceId: this.instanceId,
      serviceId: this.serviceId,
      address: await this.getLocalAddress(),
      port,
      metadata,
    });

    // 25秒ごとにハートビート（TTL 30秒の前に更新）
    this.heartbeatTimer = setInterval(async () => {
      await this.registry.heartbeat(this.instanceId).catch(err =>
        logger.error({ err }, 'Heartbeat failed')
      );
    }, 25_000);

    // グレースフルシャットダウン
    process.on('SIGTERM', () => this.stop());
    process.on('SIGINT', () => this.stop());
  }

  async stop(): Promise<void> {
    clearInterval(this.heartbeatTimer);
    await this.registry.deregister(this.instanceId, this.serviceId);
    logger.info({ instanceId: this.instanceId }, 'Service deregistered');
  }

  private async getLocalAddress(): Promise<string> {
    // Kubernetes環境: POD_IPから取得
    if (process.env.POD_IP) return process.env.POD_IP;
    // ローカル: hostnameから取得
    const { hostname } = await import('os');
    return hostname();
  }
}

// 使用例
const discovery = new ServiceDiscoveryClient(registry, 'round-robin');

// サービスに動的にHTTPリクエスト
async function callOrderService(path: string): Promise<Response> {
  const baseUrl = await discovery.resolveUrl('order-service');
  return fetch(`${baseUrl}${path}`);
}

// アプリ起動時にサービスを登録
const registrar = new ServiceRegistrar(registry, 'payment-service');
await registrar.start(3001, { version: '2.1.0', zone: 'ap-northeast-1a', weight: 1 });
```

---

## まとめ

Claude Codeでサービスディスカバリーを設計する：

1. **CLAUDE.md** にサービス名でエンドポイント解決（IPハードコード禁止）・TTL付きハートビートでゾンビ自動除去・30秒ローカルキャッシュでレジストリ負荷軽減を明記
2. **TTLベースのハートビート** でゾンビインスタンスを自動除去——`HEARTBEAT_TTL=30秒`でキーが切れたインスタンスはRedisから消える。`setInterval(heartbeat, 25_000)`で25秒ごとに更新し生存を示す
3. **クライアントサイドLB** でサービスメッシュなしに負荷分散——`round-robin`・`weighted-random`・`least-connections`を用途で選択。Istioなしで複数インスタンスに分散できる
4. **30秒ローカルキャッシュ** でレジストリへのリクエスト数を1/30に削減——毎リクエストごとにRedis参照すると自分がボトルネックになる。30秒TTLキャッシュで解決頻度を大幅削減

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
