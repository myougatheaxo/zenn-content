---
title: "Claude CodeでKubernetes HPA・オートスケーリングを設計する：負荷対応・デプロイ戦略"
emoji: "☸️"
type: "tech"
topics: ["claudecode", "kubernetes", "devops", "typescript", "aws"]
published: true
---

## はじめに

トラフィックスパイクに自動でスケールアウト——Kubernetes HPAでCPU/メモリ・カスタムメトリクスに基づく自動スケーリングを実装する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにKubernetesスケーリング設計ルールを書く

```markdown
## Kubernetes HPA設計ルール

### スケーリング設定
- minReplicas: 2（単一障害点回避）
- maxReplicas: トラフィックピーク × 1.5を上限
- CPU target: 60%（余裕を持たせる）
- スケールアップ: 即時（stabilizationWindow: 0）
- スケールダウン: 5分待機（チャーニング防止）

### カスタムメトリクス
- キューの深さでスケール（BullMQジョブ数）
- RPS（requests per second）でスケール
- Prometheus Adapterで外部メトリクス取得

### PodDisruptionBudget
- minAvailable: 1（常に1Pod以上稼働）
- ローリングアップデート: maxUnavailable 0
```

---

## Kubernetesマニフェストの生成

```
Node.jsアプリのKubernetes HPA設定を設計してください。

要件：
- Deployment（3レプリカ初期）
- HPA（CPU 60%トリガー）
- カスタムメトリクス（BullMQキュー深さ）
- PodDisruptionBudget
- ResourceQuota

生成ファイル: k8s/
```

---

## 生成されるKubernetesマニフェスト

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # ゼロダウンタイムデプロイ
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      # グレースフルシャットダウン
      terminationGracePeriodSeconds: 30
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]  # ロードバランサーが切り離す時間
        env:
        - name: NODE_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
```

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # CPUメトリクス（基本）
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60

  # メモリメトリクス
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70

  # カスタムメトリクス: BullMQキュー深さ
  - type: External
    external:
      metric:
        name: bullmq_queue_waiting_count
        selector:
          matchLabels:
            queue: "default"
      target:
        type: AverageValue
        averageValue: "10"  # Pod1つあたり10ジョブを超えたらスケール

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0  # 即時スケールアップ
      policies:
      - type: Percent
        value: 100  # 最大2倍まで一気にスケール
        periodSeconds: 60
      - type: Pods
        value: 4  # 最大4Pod増加/分
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # 5分待ってからスケールダウン
      policies:
      - type: Percent
        value: 10  # 10%ずつ減らす（急激なスケールダウン防止）
        periodSeconds: 60
```

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 1  # 常に1Pod以上稼働を保証
  selector:
    matchLabels:
      app: myapp
```

---

## カスタムメトリクスのPrometheusアダプター設定

```yaml
# k8s/prometheus-adapter-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    # BullMQキュー深さをカスタムメトリクスとして公開
    - seriesQuery: 'bullmq_queue_waiting_count{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "bullmq_queue_waiting_count"
        as: "bullmq_queue_waiting_count"
      metricsQuery: 'avg_over_time(bullmq_queue_waiting_count{<<.LabelMatchers>>}[2m])'
```

```typescript
// src/metrics.ts — Prometheusメトリクスをエクスポート
import { register, Gauge, Counter, Histogram } from 'prom-client';
import Queue from 'bull';

// BullMQキュー深さをPrometeusに公開
const queueWaitingGauge = new Gauge({
  name: 'bullmq_queue_waiting_count',
  help: 'Number of waiting jobs in BullMQ queue',
  labelNames: ['queue'],
});

// 定期的にキュー状態を更新
export async function updateQueueMetrics(queue: Queue.Queue) {
  const counts = await queue.getJobCounts();
  queueWaitingGauge.set({ queue: queue.name }, counts.waiting);
}

// HTTPリクエストメトリクス
export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

// /metricsエンドポイント
export async function metricsHandler(_req: Request, res: Response) {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
}
```

---

## グレースフルシャットダウン実装

```typescript
// src/server.ts — K8sのsigterm処理

const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Kubernetes SIGTERM対応
process.on('SIGTERM', async () => {
  console.log('SIGTERM received. Starting graceful shutdown...');

  // 新規リクエスト受付停止
  server.close(async () => {
    console.log('HTTP server closed');

    // インフライトリクエストの完了を待つ
    await Promise.all([
      prisma.$disconnect(),
      redis.quit(),
      bullQueue.close(),
    ]);

    console.log('All connections closed. Exiting.');
    process.exit(0);
  });

  // 強制終了タイムアウト（terminationGracePeriodSeconds未満）
  setTimeout(() => {
    console.error('Graceful shutdown timeout. Forcing exit.');
    process.exit(1);
  }, 25_000); // 25秒（K8sのterminationGracePeriodSeconds: 30より短く）
});
```

---

## まとめ

Claude CodeでKubernetes HPAを設計する：

1. **CLAUDE.md** にminReplicas 2・CPU 60%・スケールダウン5分待機を明記
2. **HPAのbehavior** でスケールアップは即時、スケールダウンは5分待機（チャーニング防止）
3. **カスタムメトリクス** でBullMQキュー深さをPrometheusで取得→HPAが反応
4. **PodDisruptionBudget** でminAvailable 1を保証（ノードメンテ時も停止しない）

---

*K8sスケーリング設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
