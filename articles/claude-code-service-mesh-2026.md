---
title: "Claude CodeでService Meshを設計する：Istio・mTLS・トラフィック制御・Observability"
emoji: "🕸️"
type: "tech"
topics: ["claudecode", "typescript", "kubernetes", "microservices", "devops"]
published: true
---

## はじめに

マイクロサービスが増えると、サービス間のmTLS・リトライ・タイムアウト・トレーシングを各サービスに実装するのは辛い。Service Mesh（Istio）でインフラ層に委ねる。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにService Mesh設計ルールを書く

```markdown
## Service Mesh設計ルール

### Istio活用方針
- サービス間通信のmTLSはIstioに委ねる（アプリコードに書かない）
- Retry/Timeout/Circuit BreakerはVirtualServiceで定義
- 認可ポリシーはAuthorizationPolicyで定義

### トラフィック管理
- カナリアデプロイはVirtualServiceのweight設定で制御
- フォールト注入でカオステスト（latency/abort注入）
- Ingressは istio-ingress-gateway を使う

### Observability
- Kiali でサービス間のトポロジーを可視化
- Jaeger でトレース（自動でトレースIDが伝播）
- Prometheus + Grafana でメトリクス

### 注意点
- Sidecar injectionを有効にする（namespace labelに istio-injection=enabled）
- ヘルスチェックはIstioがバイパスするよう設定
- Headless Serviceはサポート外
```

---

## Service Mesh設定の生成

```
Istioを使ったService Mesh設定を設計してください。

対象: OrderService → UserService → PaymentService
要件：
- サービス間mTLS
- OrderServiceのリトライ・タイムアウト設定
- カナリアデプロイ設定
- フォールト注入テスト

生成ファイル: k8s/istio/
```

---

## 生成されるIstio設定

```yaml
# k8s/istio/peer-authentication.yaml
# 全サービス間通信をmTLS必須に
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # mTLS必須（平文HTTP拒否）
```

```yaml
# k8s/istio/virtual-service-order.yaml
# OrderServiceのトラフィック制御
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
  namespace: production
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: canary
          weight: 100

    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90
        - destination:
            host: order-service
            subset: canary
          weight: 10

      timeout: 5s  # タイムアウト

      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: gateway-error,connect-failure,retriable-4xx

---
# k8s/istio/destination-rule-order.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
  namespace: production
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
    loadBalancer:
      simple: LEAST_CONN  # 最も接続数が少ないPodに振り分け
    outlierDetection:
      # Circuit Breaker設定
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
```

```yaml
# k8s/istio/authorization-policy.yaml
# BFFからOrderServiceへのアクセスのみ許可
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/dashboard-bff"  # BFFのサービスアカウント
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/orders*"]
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/payment-service"
      to:
        - operation:
            methods: ["PATCH"]
            paths: ["/orders/*/status"]
```

---

## フォールト注入（カオステスト）

```yaml
# k8s/istio/fault-injection.yaml
# テスト環境でUserServiceに遅延を注入
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: user-service-fault-test
  namespace: staging
spec:
  hosts:
    - user-service
  http:
    - fault:
        delay:
          percentage:
            value: 50  # 50%のリクエストに遅延
          fixedDelay: 3s  # 3秒の遅延
        abort:
          percentage:
            value: 10  # 10%のリクエストをエラー
          httpStatus: 503
      route:
        - destination:
            host: user-service
```

---

## アプリケーション側の設定（最小限）

```typescript
// Service Meshがインフラ層でリトライ・mTLS・トレーシングを処理するため
// アプリケーションコードは最小限でよい

// src/clients/userServiceClient.ts
export class UserServiceClient {
  private baseUrl = process.env.USER_SERVICE_URL!; // k8s Service DNS

  async getUser(userId: string): Promise<User> {
    // Istioが自動でmTLS、リトライ、タイムアウト、トレーシングを処理
    // アプリは普通のHTTPクライアントで呼ぶだけ
    const res = await fetch(`${this.baseUrl}/users/${userId}`, {
      headers: {
        // W3C TraceContextヘッダーをそのまま転送（Istioがトレースを繋ぐ）
        'traceparent': req.headers['traceparent'] as string,
      },
    });

    if (!res.ok) throw new HttpError(res.status, await res.text());
    return res.json();
  }
}
```

---

## まとめ

Claude CodeでService Meshを設計する：

1. **CLAUDE.md** にIstio委任方針・PeerAuthentication STRICT・AuthorizationPolicyを明記
2. **VirtualService** でリトライ・タイムアウト・カナリア重みをYAMLで管理
3. **DestinationRule** のoutlierDetectionでCircuit Breakerを設定（アプリコード不要）
4. **フォールト注入** でstaging環境のカオステスト（障害シナリオの事前検証）

---

*Service Mesh設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
