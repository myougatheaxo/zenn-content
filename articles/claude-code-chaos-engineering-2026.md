---
title: "Claude Codeでカオスエンジニアリングを設計する：障害注入・Resilience Test・本番耐障害性"
emoji: "💥"
type: "tech"
topics: ["claudecode", "devops", "kubernetes", "typescript", "testing"]
published: true
published_at: "2026-03-12 13:00"
---

## はじめに

障害が起きてから気づくのでは遅い——意図的に障害を注入して耐障害性を検証するカオスエンジニアリング。Chaos MeshとNode.jsの障害シミュレーションをClaude Codeに設計させる。

---

## CLAUDE.mdにカオスエンジニアリング設計ルールを書く

```markdown
## カオスエンジニアリング設計ルール

### 実験の原則
- まずステージング環境で実施（本番は承認後のみ）
- 爆発半径を最小化（1サービス・1リージョンから開始）
- 監視を確認してから開始（メトリクスが見えない状態で実施しない）
- 停止ボタンを用意（異常検知で即座に実験停止）

### 実験メニュー
- ネットワーク遅延（100ms・500ms・1000ms）
- パケットロス（5%・10%・20%）
- CPUスパイク（80%負荷）
- メモリプレッシャー
- Podランダム削除（Chaos Monkey）
- DBコネクション枯渇

### 成功基準
- エラー率 < 0.1%
- P99 < 500ms（遅延注入時はN/A）
- 自動フェイルオーバーが正常動作
```

---

## カオスエンジニアリングシステムの生成

```
カオスエンジニアリングフレームワークを設計してください。

要件：
- Chaos Mesh（Kubernetes上の障害注入）
- Node.jsレベルの障害シミュレーション
- 自動停止（異常検知）
- 実験レポート

生成ファイル: src/chaos/
```

---

## 生成されるカオスエンジニアリング実装

```yaml
# k8s/chaos/network-delay.yaml
# Chaos Mesh: サービス間の遅延注入
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: order-service-delay
  namespace: production
spec:
  action: delay
  mode: one             # 対象Podを1つランダムに選択
  selector:
    namespaces:
      - production
    labelSelectors:
      app: order-service
  delay:
    latency: "200ms"
    correlation: "10"   # 前のパケットとの相関
    jitter: "50ms"      # ±50msのジッター
  duration: "10m"       # 10分間実験
  direction: to         # 受信方向に遅延

---
# Podランダム削除（Chaos Monkey）
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure
  namespace: production
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - staging
    labelSelectors:
      app: myapp
  duration: "30m"
  scheduler:
    cron: "@every 5m"   # 5分ごとにランダム削除
```

```typescript
// src/chaos/chaosRunner.ts
import { CloudWatchClient, GetMetricStatisticsCommand } from '@aws-sdk/client-cloudwatch';

interface ChaosExperiment {
  id: string;
  name: string;
  description: string;
  duration: number;      // ミリ秒
  type: 'network_delay' | 'cpu_stress' | 'memory_pressure' | 'dependency_failure';
  parameters: Record<string, any>;
}

export class ChaosRunner {
  private stopRequested = false;

  async runExperiment(experiment: ChaosExperiment): Promise<ExperimentResult> {
    logger.warn({ experiment }, '⚡ Starting chaos experiment');

    const startTime = Date.now();
    const metrics: MetricSample[] = [];

    // 実験前のベースラインメトリクスを記録
    const baseline = await this.collectMetrics();

    // 実験を開始
    const experimentHandle = await this.startExperiment(experiment);

    try {
      // 実験中にメトリクスを定期収集
      const monitorInterval = setInterval(async () => {
        const sample = await this.collectMetrics();
        metrics.push(sample);

        // 異常検知: エラー率が1%超えたら即停止
        if (sample.errorRate > 0.01) {
          logger.error({ errorRate: sample.errorRate }, '🛑 High error rate detected! Stopping experiment.');
          this.stopRequested = true;
        }
      }, 10_000);

      // 実験時間が経過するか停止要求があるまで待機
      await new Promise<void>((resolve) => {
        const timer = setTimeout(resolve, experiment.duration);
        const checker = setInterval(() => {
          if (this.stopRequested) {
            clearTimeout(timer);
            clearInterval(checker);
            resolve();
          }
        }, 1_000);
      });

      clearInterval(monitorInterval);
    } finally {
      // 必ず実験を停止
      await this.stopExperiment(experimentHandle);
      this.stopRequested = false;
    }

    const result: ExperimentResult = {
      experimentId: experiment.id,
      durationMs: Date.now() - startTime,
      baseline,
      metrics,
      conclusion: this.analyzeResults(baseline, metrics),
    };

    await this.saveResult(result);
    return result;
  }

  private analyzeResults(baseline: MetricSample, samples: MetricSample[]): Conclusion {
    const avgErrorRate = samples.reduce((s, m) => s + m.errorRate, 0) / samples.length;
    const p99 = samples.reduce((max, m) => Math.max(max, m.p99LatencyMs), 0);
    const recovered = samples[samples.length - 1].errorRate < 0.001;

    return {
      passed: avgErrorRate < 0.001 && recovered,
      findings: [
        avgErrorRate > baseline.errorRate * 5
          ? `Error rate increased ${(avgErrorRate / baseline.errorRate).toFixed(1)}x during experiment`
          : 'Error rate remained stable',
        `Peak P99 latency: ${p99}ms`,
        recovered ? 'Service recovered automatically' : '⚠️ Service did not fully recover',
      ],
    };
  }
}
```

```typescript
// src/chaos/faultInjection.ts
// Node.jsレベルの障害シミュレーション（依存サービスの障害テスト）

// HTTPクライアントに障害を注入するラッパー
export function injectFaults(config: {
  errorRate: number;     // 0-1（0.1 = 10%エラー）
  latencyMs?: number;    // 追加遅延
  timeout?: boolean;     // タイムアウトを発生させる
}) {
  return async (url: string, options: RequestInit): Promise<Response> => {
    // 遅延注入
    if (config.latencyMs) {
      await new Promise(r => setTimeout(r, config.latencyMs));
    }

    // タイムアウト注入
    if (config.timeout && Math.random() < config.errorRate) {
      throw new Error('Injected timeout: request exceeded time limit');
    }

    // エラー注入
    if (Math.random() < config.errorRate) {
      throw new Error('Injected fault: simulated service failure');
    }

    return fetch(url, options);
  };
}

// 使用例: サードパーティAPIの障害をシミュレート
export const testPaymentServiceResilience = async () => {
  const faultyFetch = injectFaults({ errorRate: 0.3, latencyMs: 500 });

  // Circuit breakerが正常に動作するかテスト
  const results = await Promise.allSettled(
    Array.from({ length: 10 }, () =>
      callPaymentService({ fetch: faultyFetch })
    )
  );

  const failures = results.filter(r => r.status === 'rejected');
  console.log(`Circuit breaker test: ${failures.length}/10 failed (expected ~3)`);
};
```

---

## まとめ

Claude Codeでカオスエンジニアリングを設計する：

1. **CLAUDE.md** にステージ先実施・爆発半径最小・監視確認後実施・停止ボタン必須を明記
2. **Chaos Mesh** でネットワーク遅延・Pod削除をKubernetesレベルで注入
3. **エラー率1%超えで自動停止** ——実験が本番影響を超えたら即座に中断
4. **faultInjection** でNode.jsレベルのテスト——CI環境でCircuit Breakerの動作を検証

---

*カオスエンジニアリング設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
