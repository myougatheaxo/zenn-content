---
title: "Claude CodeでNode.jsパフォーマンスプロファイリングを設計する：CPUフレームグラフ・メモリ分析・ボトルネック特定"
emoji: "🔥"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture"]
published: true
published_at: "2026-03-21 15:00"
---

## はじめに

「APIが遅いけどどこがボトルネックか分からない」「メモリが増え続けるけどリークの場所が特定できない」——CPUフレームグラフ・ヒープスナップショット・Clinic.jsでNode.jsのパフォーマンス問題を特定する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにパフォーマンスプロファイリング設計ルールを書く

```markdown
## Node.jsパフォーマンスプロファイリング設計ルール

### ツールチェーン
- clinic.js: 統合プロファイラー（Doctor/Flame/Bubbleprof）
- 0x: CPUフレームグラフ生成（Node.js組み込みV8プロファイラー）
- node --inspect: Chrome DevToolsでリアルタイムプロファイル

### ボトルネック別診断
- CPU使用率高い: 0x + フレームグラフで同期処理のホットパスを特定
- レイテンシ高い: clinic bubbleprof で非同期の待ち時間を分析
- メモリリーク: clinic heapprofile + heapdump でヒープ分析

### 本番での継続監視
- process.memoryUsage() を定期記録
- Performance API (perf_hooks) でクリティカルパスを計測
- prom-client でカスタムメトリクスをPrometheusに送信
```

---

## パフォーマンスプロファイリング実装の生成

```
Node.jsパフォーマンスプロファイリングを設計してください。

要件：
- 本番対応のパフォーマンス計測
- メモリリーク検出
- カスタムメトリクス収集
- ホットパス最適化

生成ファイル: src/monitoring/
```

---

## 生成されるパフォーマンスプロファイリング実装

```typescript
// src/monitoring/performanceTracker.ts — パフォーマンス計測

import { performance, PerformanceObserver } from 'perf_hooks';

// 関数実行時間をPerformance APIで計測
export function measurePerformance<T>(
  name: string,
  fn: () => T | Promise<T>
): Promise<{ result: T; durationMs: number }> {
  return new Promise(async (resolve, reject) => {
    const start = performance.now();
    const markStart = `${name}-start`;
    const markEnd = `${name}-end`;

    performance.mark(markStart);
    try {
      const result = await fn();
      performance.mark(markEnd);
      performance.measure(name, markStart, markEnd);

      const durationMs = performance.now() - start;
      resolve({ result, durationMs });
    } catch (error) {
      performance.mark(markEnd);
      reject(error);
    } finally {
      // マークをクリーンアップ
      performance.clearMarks(markStart);
      performance.clearMarks(markEnd);
    }
  });
}

// PerformanceObserverでメジャーを収集
export function setupPerformanceMonitoring(): void {
  const obs = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (entry.duration > 100) {  // 100ms以上の計測を記録
        logger.warn({
          name: entry.name,
          durationMs: entry.duration.toFixed(2),
        }, 'Slow operation detected');

        // Prometheusメトリクスに記録
        operationHistogram.observe({ operation: entry.name }, entry.duration);
      }
    }
  });

  obs.observe({ entryTypes: ['measure'] });
}

// Express ミドルウェア: リクエストのP99計測
export const performanceMiddleware = (req: Request, res: Response, next: NextFunction): void => {
  const start = performance.now();

  res.on('finish', () => {
    const durationMs = performance.now() - start;
    const route = req.route?.path ?? req.path;

    // ヒストグラムに記録（P50/P95/P99を計算）
    httpRequestHistogram.observe(
      { method: req.method, route, status: String(res.statusCode) },
      durationMs
    );

    // 遅いリクエストをログ
    if (durationMs > 1000) {
      logger.warn({ method: req.method, route, durationMs, status: res.statusCode }, 'Slow request');
    }
  });

  next();
};
```

```typescript
// src/monitoring/memoryProfiler.ts — メモリリーク検出

import v8 from 'v8';
import { writeFileSync } from 'fs';

export class MemoryProfiler {
  private baselineHeap: number = 0;
  private samples: Array<{ timestamp: Date; heapUsed: number; heapTotal: number; external: number }> = [];

  // ベースラインを記録（起動直後に呼ぶ）
  setBaseline(): void {
    const { heapUsed } = process.memoryUsage();
    this.baselineHeap = heapUsed;
    logger.info({ baselineHeapMB: (heapUsed / 1024 / 1024).toFixed(2) }, 'Memory baseline set');
  }

  // 定期サンプリング（メモリリーク傾向を検出）
  startSampling(intervalMs: number = 30_000): NodeJS.Timeout {
    return setInterval(() => {
      const mem = process.memoryUsage();
      const sample = {
        timestamp: new Date(),
        heapUsed: mem.heapUsed,
        heapTotal: mem.heapTotal,
        external: mem.external,
      };

      this.samples.push(sample);

      // 直近10サンプルが継続増加していたらリーク警告
      if (this.samples.length >= 10) {
        const recent = this.samples.slice(-10);
        const isIncreasing = recent.every((s, i) =>
          i === 0 || s.heapUsed > recent[i - 1].heapUsed
        );

        if (isIncreasing) {
          const growthMB = (recent[9].heapUsed - recent[0].heapUsed) / 1024 / 1024;
          logger.error({
            growthMB: growthMB.toFixed(2),
            heapUsedMB: (mem.heapUsed / 1024 / 1024).toFixed(2),
          }, 'Potential memory leak detected - heap consistently growing');

          // 自動ヒープダンプ（開発環境のみ）
          if (process.env.NODE_ENV === 'development') {
            this.takeHeapSnapshot();
          }
        }
      }

      // メトリクス送信
      memoryGauge.set({ type: 'heap_used' }, mem.heapUsed);
      memoryGauge.set({ type: 'heap_total' }, mem.heapTotal);
      memoryGauge.set({ type: 'external' }, mem.external);

      // サンプル数を制限
      if (this.samples.length > 100) {
        this.samples.shift();
      }
    }, intervalMs);
  }

  // ヒープスナップショットを保存（Chrome DevToolsで分析）
  takeHeapSnapshot(): string {
    const snapshotPath = `/tmp/heap-snapshot-${Date.now()}.heapsnapshot`;
    const snapshotStream = v8.writeHeapSnapshot(snapshotPath);
    logger.info({ path: snapshotPath }, 'Heap snapshot written');
    return snapshotPath;
  }

  // メモリ使用量サマリー
  getSummary(): {
    currentMB: number;
    baselineMB: number;
    growthMB: number;
    gcCount: number;
  } {
    const { heapUsed } = process.memoryUsage();
    return {
      currentMB: heapUsed / 1024 / 1024,
      baselineMB: this.baselineHeap / 1024 / 1024,
      growthMB: (heapUsed - this.baselineHeap) / 1024 / 1024,
      gcCount: 0,  // GCイベントはnode --expose-gc で取得
    };
  }
}

// CLI分析コマンド（package.jsonに追加）
// "profile:cpu": "0x --output-dir .profiles/cpu node dist/main.js"
// "profile:memory": "node --heapsnapshot-near-heap-limit=5 dist/main.js"
// "profile:clinic": "clinic doctor -- node dist/main.js"

// ボトルネック最適化例
// ❌ 遅い実装（同期的な重い処理）
function processLargeDataSync(data: number[]): number {
  return data.reduce((sum, n) => sum + Math.sqrt(n), 0);
}

// ✅ 最適化（チャンク分割で非同期化）
async function processLargeDataAsync(data: number[]): Promise<number> {
  const CHUNK_SIZE = 1000;
  let total = 0;

  for (let i = 0; i < data.length; i += CHUNK_SIZE) {
    const chunk = data.slice(i, i + CHUNK_SIZE);
    total += chunk.reduce((sum, n) => sum + Math.sqrt(n), 0);

    // イベントループを解放（他のリクエストを処理させる）
    if (i + CHUNK_SIZE < data.length) {
      await new Promise(resolve => setImmediate(resolve));
    }
  }

  return total;
}
```

---

## まとめ

Claude CodeでNode.jsパフォーマンスプロファイリングを設計する：

1. **CLAUDE.md** にCPU問題は0x+フレームグラフ・非同期レイテンシはclinic bubbleprof・メモリリークはヒープスナップショット+継続サンプリング・本番はprom-clientでカスタムメトリクスを明記
2. **Performance APIで重要パスを計測** ——`performance.mark()`→`performance.measure()`でコードブロックの実行時間をナノ秒精度で計測。PerformanceObserverで100ms超の操作を自動ログ
3. **ヒープ継続サンプリング** でメモリリークを早期検知——30秒ごとにヒープ使用量を記録し、10連続増加でリーク警告。開発環境では自動でheapsnaphotを生成してChrome DevToolsで分析できる
4. **重い同期処理を`setImmediate()`でチャンク分割** ——100万件のデータ処理を1000件ずつ分割し、チャンク間に`setImmediate()`を挿入。イベントループを解放して他のリクエストの処理を妨げない

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
