---
title: "Claude CodeでNode.js Worker Threadsを設計する：CPU集約タスクの並列化・スレッドプール"
emoji: "⚙️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "performance"]
published: true
published_at: "2026-03-14 13:00"
---

## はじめに

Node.jsのシングルスレッドでCPU集約処理をすると他のリクエストがブロックされる——`worker_threads`で重い処理を別スレッドに分離し、スレッドプールで効率的に管理する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにWorker Threads設計ルールを書く

```markdown
## Worker Threads設計ルール

### スレッドプール設計
- CPU数に合わせたスレッド数: os.cpus().length - 1（メインスレッド用に1残す）
- アイドル時のメモリ節約: 最小スレッド数 = 2
- タスクキューが溢れた場合: RejectedError（バックプレッシャー）

### 対象処理
- 画像処理・動画変換・PDFレンダリング
- 暗号化・圧縮・解凍（zlibの同期版）
- 大量データのCSV/JSON変換
- ML推論（TensorFlow.js等）

### 禁止パターン
- Workerからのイベントループブロック（fs.readFileSync等）
- SharedArrayBufferの無制御アクセス（Atomicsを使うこと）
- 大量データのstructuredClone（postMessageのコピーコスト）
```

---

## Worker Threads実装の生成

```
Node.js worker_threadsによるスレッドプールを設計してください。

要件：
- CPUコア数に最適化したスレッドプール
- タスクキュー管理
- タイムアウト処理
- エラーハンドリング

生成ファイル: src/workers/
```

---

## 生成されるWorker Threads実装

```typescript
// src/workers/threadPool.ts — スレッドプール管理

import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';
import { cpus } from 'os';
import { EventEmitter } from 'events';

interface TaskItem<T, R> {
  data: T;
  resolve: (result: R) => void;
  reject: (error: Error) => void;
  timeoutHandle?: NodeJS.Timeout;
}

export class ThreadPool<TInput = unknown, TOutput = unknown> {
  private workers: Worker[] = [];
  private idleWorkers: Worker[] = [];
  private taskQueue: TaskItem<TInput, TOutput>[] = [];
  private workerTaskMap = new Map<Worker, TaskItem<TInput, TOutput>>();

  private readonly minThreads: number;
  private readonly maxThreads: number;
  private readonly taskTimeoutMs: number;
  private readonly maxQueueSize: number;

  constructor(
    private readonly workerScript: string,
    options: {
      minThreads?: number;
      maxThreads?: number;
      taskTimeoutMs?: number;
      maxQueueSize?: number;
    } = {}
  ) {
    const cpuCount = cpus().length;
    this.minThreads = options.minThreads ?? 2;
    this.maxThreads = options.maxThreads ?? Math.max(cpuCount - 1, 1);
    this.taskTimeoutMs = options.taskTimeoutMs ?? 30_000;
    this.maxQueueSize = options.maxQueueSize ?? 100;

    // 最小スレッド数で初期化
    for (let i = 0; i < this.minThreads; i++) {
      this.addWorker();
    }

    logger.info({
      minThreads: this.minThreads,
      maxThreads: this.maxThreads,
      cpuCount,
    }, 'Thread pool initialized');
  }

  // タスクをスレッドプールに投入
  async run(data: TInput): Promise<TOutput> {
    if (this.taskQueue.length >= this.maxQueueSize) {
      throw new Error(`Thread pool queue full (max: ${this.maxQueueSize})`);
    }

    return new Promise<TOutput>((resolve, reject) => {
      const task: TaskItem<TInput, TOutput> = { data, resolve, reject };

      // アイドルワーカーがあれば即実行
      const idleWorker = this.idleWorkers.pop();
      if (idleWorker) {
        this.runTaskOnWorker(idleWorker, task);
        return;
      }

      // スレッドを追加できる場合は追加
      if (this.workers.length < this.maxThreads) {
        const worker = this.addWorker();
        this.runTaskOnWorker(worker, task);
        return;
      }

      // キューに積む（タイムアウト付き）
      task.timeoutHandle = setTimeout(() => {
        const idx = this.taskQueue.indexOf(task);
        if (idx !== -1) {
          this.taskQueue.splice(idx, 1);
          reject(new Error(`Task timed out after ${this.taskTimeoutMs}ms in queue`));
        }
      }, this.taskTimeoutMs);

      this.taskQueue.push(task);
    });
  }

  private addWorker(): Worker {
    const worker = new Worker(this.workerScript);

    worker.on('message', (result: { success: true; data: TOutput } | { success: false; error: string }) => {
      const task = this.workerTaskMap.get(worker);
      if (!task) return;

      this.workerTaskMap.delete(worker);
      if (task.timeoutHandle) clearTimeout(task.timeoutHandle);

      if (result.success) {
        task.resolve(result.data);
      } else {
        task.reject(new Error(result.error));
      }

      // 次のタスクを処理
      const nextTask = this.taskQueue.shift();
      if (nextTask) {
        if (nextTask.timeoutHandle) clearTimeout(nextTask.timeoutHandle);
        this.runTaskOnWorker(worker, nextTask);
      } else {
        this.idleWorkers.push(worker);

        // 最小スレッド数を超えるアイドルワーカーは終了
        if (this.idleWorkers.length > this.minThreads) {
          const excess = this.idleWorkers.pop()!;
          this.terminateWorker(excess);
        }
      }
    });

    worker.on('error', (err) => {
      const task = this.workerTaskMap.get(worker);
      this.terminateWorker(worker);
      if (task) task.reject(err);
      // 最小スレッド数を維持するため新しいワーカーを追加
      if (this.workers.length < this.minThreads) this.addWorker();
    });

    this.workers.push(worker);
    this.idleWorkers.push(worker);
    return worker;
  }

  private runTaskOnWorker(worker: Worker, task: TaskItem<TInput, TOutput>): void {
    this.workerTaskMap.set(worker, task);
    const idxInIdle = this.idleWorkers.indexOf(worker);
    if (idxInIdle !== -1) this.idleWorkers.splice(idxInIdle, 1);

    // タスクタイムアウト
    task.timeoutHandle = setTimeout(() => {
      this.terminateWorker(worker);
      task.reject(new Error(`Worker task timed out after ${this.taskTimeoutMs}ms`));
      if (this.workers.length < this.minThreads) this.addWorker();
    }, this.taskTimeoutMs);

    worker.postMessage(task.data);
  }

  private terminateWorker(worker: Worker): void {
    const idx = this.workers.indexOf(worker);
    if (idx !== -1) this.workers.splice(idx, 1);
    this.workerTaskMap.delete(worker);
    worker.terminate();
  }

  async shutdown(): Promise<void> {
    await Promise.all(this.workers.map(w => w.terminate()));
    this.workers = [];
    this.idleWorkers = [];
  }

  get stats() {
    return {
      total: this.workers.length,
      idle: this.idleWorkers.length,
      busy: this.workers.length - this.idleWorkers.length,
      queued: this.taskQueue.length,
    };
  }
}
```

```typescript
// src/workers/imageProcessor.worker.ts — 画像処理ワーカー

import { parentPort } from 'worker_threads';
import sharp from 'sharp';

interface ImageTask {
  buffer: Buffer;
  width: number;
  height: number;
  format: 'webp' | 'jpeg' | 'png';
  quality: number;
}

// メインスレッドからのメッセージを処理
parentPort!.on('message', async (task: ImageTask) => {
  try {
    const result = await sharp(task.buffer)
      .resize(task.width, task.height, { fit: 'inside', withoutEnlargement: true })
      .toFormat(task.format, { quality: task.quality })
      .toBuffer();

    parentPort!.postMessage({ success: true, data: result });
  } catch (err) {
    parentPort!.postMessage({ success: false, error: (err as Error).message });
  }
});

// src/workers/index.ts — スレッドプールのシングルトン
export const imagePool = new ThreadPool<ImageTask, Buffer>(
  path.join(__dirname, 'imageProcessor.worker.js'),
  { minThreads: 2, maxThreads: 4, taskTimeoutMs: 30_000 }
);

// 使用例
export async function processImage(
  buffer: Buffer,
  options: { width: number; height: number }
): Promise<Buffer> {
  return imagePool.run({
    buffer,
    width: options.width,
    height: options.height,
    format: 'webp',
    quality: 80,
  });
}
```

---

## まとめ

Claude CodeでNode.js Worker Threadsを設計する：

1. **CLAUDE.md** にCPU数-1スレッド・タスクキュー上限・タイムアウト・SharedArrayBuffer禁止を明記
2. **アイドルワーカー管理** でアイドル数が最小値を超えた場合にスレッドを自動終了——メモリ節約
3. **二重タイムアウト** でキュー内タイムアウト（待機時間）と実行タイムアウト（処理時間）の両方を管理
4. **ワーカーエラー時** は自動で新しいワーカーを補充——最小スレッド数を常に維持

---

*パフォーマンス設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
