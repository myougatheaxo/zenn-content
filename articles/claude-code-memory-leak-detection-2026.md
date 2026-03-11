---
title: "Claude CodeでNode.jsメモリリークを検出する：heapdump・clinic.js・V8フレームグラフ"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "performance", "observability"]
published: true
published_at: "2026-03-13 07:00"
---

## はじめに

「デプロイ後数時間でメモリが倍になる」——Node.jsのメモリリークをheapdump・clinic.js・V8プロファイラで特定し、自動アラートで早期検出する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにメモリリーク検出設計ルールを書く

```markdown
## メモリリーク検出設計ルール

### 計測指標
- process.memoryUsage().heapUsed をPrometheusに記録（15秒間隔）
- ヒープ使用率が80%超 → Slackアラート
- 1時間で20%以上増加 → 重大アラート（リークの可能性）

### 定期ヒープスナップショット
- 本番: 1時間ごとに heapdump を /tmp に保存（24時間で削除）
- ステージング: 開発時は --inspect フラグでChrome DevTools接続
- OOM発生時: --heapdump-on-oom で自動ダンプ

### 代表的なリークパターン
- グローバル変数への無限追加
- イベントリスナーの削除忘れ
- キャッシュの上限なし拡張
- クロージャによる外部参照保持
```

---

## メモリリーク検出システムの生成

```
Node.jsアプリのメモリリーク検出システムを設計してください。

要件：
- ヒープ使用量のPrometheus計測
- 自動スナップショット
- リーク自動アラート
- よくあるリークパターンの検出

生成ファイル: src/monitoring/
```

---

## 生成されるメモリリーク検出実装

```typescript
// src/monitoring/memoryMonitor.ts

import { Gauge, Counter } from 'prom-client';

// Prometheusメトリクス登録
const heapUsed = new Gauge({
  name: 'nodejs_heap_used_bytes',
  help: 'Node.js heap used in bytes',
});

const heapTotal = new Gauge({
  name: 'nodejs_heap_total_bytes',
  help: 'Node.js heap total in bytes',
});

const externalMemory = new Gauge({
  name: 'nodejs_external_memory_bytes',
  help: 'Node.js external memory (C++ objects) in bytes',
});

const heapSpaces = new Gauge({
  name: 'nodejs_heap_space_used_bytes',
  help: 'Node.js heap space used bytes',
  labelNames: ['space'],
});

const gcRunsTotal = new Counter({
  name: 'nodejs_gc_runs_total',
  help: 'Number of GC runs',
  labelNames: ['type'],
});

// ヒープ使用状況スナップショット（リーク検出用）
const heapHistory: Array<{ timestamp: number; heapUsed: number }> = [];
const MAX_HISTORY = 12; // 3時間分（15分×12）

export function startMemoryMonitoring(): void {
  // 15秒ごとにメトリクス更新
  setInterval(() => {
    const mem = process.memoryUsage();
    heapUsed.set(mem.heapUsed);
    heapTotal.set(mem.heapTotal);
    externalMemory.set(mem.external);

    // ヒープ空間ごとの詳細
    const v8 = require('v8');
    for (const space of v8.getHeapSpaceStatistics()) {
      heapSpaces.set({ space: space.space_name }, space.space_used_size);
    }

    // 増加傾向チェック（15分×12=3時間分を保持）
    heapHistory.push({ timestamp: Date.now(), heapUsed: mem.heapUsed });
    if (heapHistory.length > MAX_HISTORY) heapHistory.shift();

    checkForLeak(mem);
  }, 15_000);

  // GCイベントを追跡
  if (process.env.NODE_ENV === 'production') {
    try {
      const gcProfiler = require('gc-profiler');
      gcProfiler.on('gc', ({ type }: { type: string }) => {
        gcRunsTotal.inc({ type });
      });
    } catch {
      // gc-profilerが無ければスキップ
    }
  }

  logger.info('Memory monitoring started');
}

function checkForLeak(mem: NodeJS.MemoryUsage): void {
  const heapPercent = (mem.heapUsed / mem.heapTotal) * 100;

  // ヒープ80%超アラート
  if (heapPercent > 80) {
    logger.warn({
      heapUsedMB: Math.round(mem.heapUsed / 1024 / 1024),
      heapTotalMB: Math.round(mem.heapTotal / 1024 / 1024),
      percent: heapPercent.toFixed(1),
    }, 'High heap usage detected');

    Sentry.captureMessage('High Node.js heap usage', {
      level: 'warning',
      extra: { heapUsed: mem.heapUsed, heapTotal: mem.heapTotal },
    });
  }

  // 1時間で20%以上増加チェック（12サンプル以上ある場合）
  if (heapHistory.length >= 12) {
    const oldest = heapHistory[0].heapUsed;
    const current = mem.heapUsed;
    const growthRate = ((current - oldest) / oldest) * 100;

    if (growthRate > 20) {
      logger.error({
        oldestMB: Math.round(oldest / 1024 / 1024),
        currentMB: Math.round(current / 1024 / 1024),
        growthPercent: growthRate.toFixed(1),
      }, 'Possible memory leak: heap grew 20%+ in 3 hours');

      Sentry.captureMessage('Possible memory leak detected', {
        level: 'error',
        extra: { growthRate, heapUsed: current, oldest },
      });

      sendSlackAlert(`🚨 Memory Leak Detected: Heap grew ${growthRate.toFixed(1)}% in 3 hours (${Math.round(oldest/1024/1024)}MB → ${Math.round(current/1024/1024)}MB)`);
    }
  }
}
```

```typescript
// src/monitoring/heapSnapshot.ts — 定期ヒープスナップショット

import { writeHeapSnapshot } from 'v8';
import { unlink, readdir, stat } from 'fs/promises';
import path from 'path';

const SNAPSHOT_DIR = '/tmp/heapsnapshots';
const MAX_SNAPSHOTS = 24; // 24時間分

export function scheduleHeapSnapshots(): void {
  if (process.env.HEAP_SNAPSHOTS !== 'true') return;

  // 1時間ごとにスナップショット
  setInterval(async () => {
    try {
      const filename = writeHeapSnapshot(SNAPSHOT_DIR);
      logger.info({ filename }, 'Heap snapshot written');

      // 古いスナップショットを削除
      await cleanupOldSnapshots();
    } catch (err) {
      logger.error({ err }, 'Failed to write heap snapshot');
    }
  }, 3_600_000);
}

async function cleanupOldSnapshots(): Promise<void> {
  const files = await readdir(SNAPSHOT_DIR);
  const snapshots = files
    .filter(f => f.endsWith('.heapsnapshot'))
    .sort();

  if (snapshots.length > MAX_SNAPSHOTS) {
    const toDelete = snapshots.slice(0, snapshots.length - MAX_SNAPSHOTS);
    await Promise.all(toDelete.map(f => unlink(path.join(SNAPSHOT_DIR, f))));
    logger.debug({ deleted: toDelete.length }, 'Old heap snapshots cleaned up');
  }
}

// オンデマンドスナップショット（デバッグ用エンドポイント）
// 本番では管理者のみアクセス可能にすること
export function heapSnapshotRouter(): Router {
  const router = express.Router();

  router.post('/debug/heap-snapshot', requireAdmin, async (req, res) => {
    const filename = writeHeapSnapshot(SNAPSHOT_DIR);
    res.json({ filename, message: 'Heap snapshot written' });
  });

  return router;
}
```

```typescript
// src/monitoring/leakDetector.ts — よくあるリークパターンの自動検出

export class LeakDetector {
  private readonly checks: Array<() => LeakCheckResult> = [];

  // グローバルキャッシュのサイズ監視
  watchCache(name: string, cache: Map<unknown, unknown>, maxSize = 1000): void {
    this.checks.push(() => {
      if (cache.size > maxSize) {
        return {
          detected: true,
          name: `cache:${name}`,
          message: `Cache "${name}" has ${cache.size} entries (max: ${maxSize})`,
          severity: cache.size > maxSize * 2 ? 'error' : 'warn',
        };
      }
      return { detected: false, name: `cache:${name}` };
    });
  }

  // EventEmitterのリスナー数監視
  watchEventEmitter(name: string, emitter: EventEmitter, maxListeners = 20): void {
    this.checks.push(() => {
      const listenerCount = emitter.eventNames().reduce(
        (sum, event) => sum + emitter.listenerCount(event as string), 0
      );
      if (listenerCount > maxListeners) {
        return {
          detected: true,
          name: `emitter:${name}`,
          message: `EventEmitter "${name}" has ${listenerCount} listeners (max: ${maxListeners})`,
          severity: 'warn',
        };
      }
      return { detected: false, name: `emitter:${name}` };
    });
  }

  // 定期チェック実行
  startChecking(intervalMs = 60_000): void {
    setInterval(() => {
      for (const check of this.checks) {
        const result = check();
        if (result.detected) {
          logger[result.severity ?? 'warn']({ name: result.name }, result.message);
        }
      }
    }, intervalMs);
  }
}

// 使用例
export const leakDetector = new LeakDetector();

// アプリ初期化時に登録
leakDetector.watchCache('userSessions', sessionCache, 10000);
leakDetector.watchCache('queryResults', queryCache, 5000);
leakDetector.watchEventEmitter('httpServer', httpServer, 30);
leakDetector.startChecking();
```

---

## まとめ

Claude CodeでNode.jsメモリリークを検出する：

1. **CLAUDE.md** にヒープ80%でアラート・3時間20%増加でリーク疑い・1時間ごとheapdumpを明記
2. **heapHistory配列** で過去3時間のヒープ推移を保持し増加率を自動計算
3. **LeakDetector** でMapキャッシュのサイズ・EventEmitterのリスナー数を定期監視
4. **gc-profiler** でGC頻度を記録——GCが増えても使用量が減らない場合はリーク確定

---

*パフォーマンス診断のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
