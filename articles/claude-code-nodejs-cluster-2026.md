---
title: "Claude CodeでNode.jsクラスターモードを設計する：マルチプロセス・ゼロダウンタイムリロード・プロセス間通信"
emoji: "⚙️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture"]
published: true
published_at: "2026-03-21 11:00"
---

## はじめに

「Node.jsがシングルプロセスでCPUが1コアしか使われない」「デプロイのたびにサービスが数秒停止する」——Node.jsのclusterモジュールで複数プロセスを起動し、ゼロダウンタイムリロードとプロセス間通信を設計をClaude Codeに生成させる。

---

## CLAUDE.mdにNode.jsクラスター設計ルールを書く

```markdown
## Node.jsクラスター設計ルール

### プロセス構成
- Masterプロセス: ワーカー管理のみ（HTTPは処理しない）
- Workerプロセス: HTTPリクエスト処理（CPU数分起動）
- ワーカー数: os.cpus().length（コンテナならCPU制限に応じて調整）

### ゼロダウンタイムリロード
- ワーカーを1台ずつ再起動（ローリングリストア）
- 新ワーカーが起動してから旧ワーカーをシャットダウン
- SIGUSR2シグナルでリロードをトリガー

### プロセス間通信（IPC）
- Masterがメッセージをブロードキャスト
- ワーカー間で直接通信しない（Master経由）
- 共有状態はRedisに保存（プロセスローカルメモリに保存しない）
```

---

## Node.jsクラスター実装の生成

```
Node.jsクラスターモードを設計してください。

要件：
- クラスターマスター
- ゼロダウンタイムリロード
- ワーカーの自動再起動
- プロセス間通信

生成ファイル: src/cluster/
```

---

## 生成されるNode.jsクラスター実装

```typescript
// src/cluster/master.ts — クラスターマスター

import cluster, { Worker } from 'cluster';
import os from 'os';

export interface ClusterConfig {
  workerCount?: number;        // デフォルト: CPU数
  reloadSignal?: NodeJS.Signals;  // デフォルト: SIGUSR2
  restartDelay?: number;       // ワーカー再起動の待機時間（ms）
  maxRestartAttempts?: number; // ワーカーの最大再起動回数
}

export class ClusterMaster {
  private readonly workerCount: number;
  private readonly workers = new Map<number, {
    worker: Worker;
    restartCount: number;
    startedAt: Date;
  }>();
  private isShuttingDown = false;

  constructor(private readonly config: ClusterConfig = {}) {
    this.workerCount = config.workerCount ?? os.cpus().length;
  }

  start(): void {
    if (!cluster.isPrimary) {
      throw new Error('ClusterMaster must be run in primary process');
    }

    logger.info({ workerCount: this.workerCount }, 'Starting cluster');

    // 指定数のワーカーを起動
    for (let i = 0; i < this.workerCount; i++) {
      this.spawnWorker();
    }

    // ワーカーの異常終了を監視
    cluster.on('exit', (worker, code, signal) => {
      const meta = this.workers.get(worker.id);
      this.workers.delete(worker.id);

      if (this.isShuttingDown) return;

      logger.warn({ workerId: worker.id, code, signal }, 'Worker died');

      const maxRestarts = this.config.maxRestartAttempts ?? 10;
      if ((meta?.restartCount ?? 0) >= maxRestarts) {
        logger.error({ workerId: worker.id }, 'Worker exceeded max restart attempts');
        return;
      }

      // ワーカーを再起動
      setTimeout(() => {
        if (!this.isShuttingDown) {
          this.spawnWorker(meta?.restartCount);
        }
      }, this.config.restartDelay ?? 1000);
    });

    // ゼロダウンタイムリロード（SIGUSR2でトリガー）
    process.on(this.config.reloadSignal ?? 'SIGUSR2', () => {
      this.rollingReload().catch(err =>
        logger.error({ err }, 'Rolling reload failed')
      );
    });

    // グレースフルシャットダウン
    process.on('SIGTERM', () => this.gracefulShutdown());
    process.on('SIGINT', () => this.gracefulShutdown());

    // プロセス間通信
    cluster.on('message', (worker, message) => {
      this.handleWorkerMessage(worker, message);
    });

    logger.info({ workerCount: this.workerCount }, 'Cluster started');
  }

  private spawnWorker(previousRestartCount = 0): Worker {
    const worker = cluster.fork({
      WORKER_ID: String(cluster.workers ? Object.keys(cluster.workers).length : 0),
    });

    this.workers.set(worker.id, {
      worker,
      restartCount: previousRestartCount + 1,
      startedAt: new Date(),
    });

    worker.on('online', () => {
      logger.info({ workerId: worker.id }, 'Worker online');
    });

    return worker;
  }

  // ゼロダウンタイムローリングリロード
  async rollingReload(): Promise<void> {
    logger.info('Starting rolling reload');
    const workerIds = [...this.workers.keys()];

    for (const workerId of workerIds) {
      const meta = this.workers.get(workerId);
      if (!meta) continue;

      // 新ワーカーを先に起動
      const newWorker = this.spawnWorker(0);

      // 新ワーカーがオンラインになるまで待機
      await new Promise<void>((resolve) => {
        newWorker.on('online', resolve);
        setTimeout(resolve, 5000);  // タイムアウト
      });

      // 旧ワーカーにグレースフルシャットダウンを送信
      meta.worker.send({ type: 'graceful-shutdown' });

      // ワーカーが終了するまで待機
      await new Promise<void>((resolve) => {
        meta.worker.once('exit', resolve);
        setTimeout(() => {
          meta.worker.kill('SIGTERM');
          resolve();
        }, 30_000);  // 30秒でタイムアウト
      });

      logger.info({ workerId, newWorkerId: newWorker.id }, 'Worker replaced');

      // 次のワーカーをリロードする前に少し待つ
      await sleep(500);
    }

    logger.info('Rolling reload completed');
  }

  // マスターからワーカーへのブロードキャスト
  broadcast(message: unknown): void {
    for (const { worker } of this.workers.values()) {
      if (worker.isConnected()) {
        worker.send(message);
      }
    }
  }

  // ワーカーからのメッセージハンドラー
  private handleWorkerMessage(worker: Worker, message: any): void {
    switch (message.type) {
      case 'broadcast':
        // あるワーカーから受け取ったメッセージを全ワーカーに転送
        for (const [id, meta] of this.workers) {
          if (id !== worker.id && meta.worker.isConnected()) {
            meta.worker.send({ ...message, fromWorkerId: worker.id });
          }
        }
        break;
      case 'metrics':
        metricsAggregator.record(worker.id, message.data);
        break;
    }
  }

  private async gracefulShutdown(): Promise<void> {
    if (this.isShuttingDown) return;
    this.isShuttingDown = true;
    logger.info('Master graceful shutdown');

    this.broadcast({ type: 'graceful-shutdown' });

    // 全ワーカーの終了を待機（最大30秒）
    await Promise.race([
      Promise.all([...this.workers.values()].map(
        ({ worker }) => new Promise<void>(resolve => worker.once('exit', resolve))
      )),
      sleep(30_000),
    ]);

    process.exit(0);
  }
}
```

```typescript
// src/cluster/worker.ts — クラスターワーカー

export function startWorker(app: Express): void {
  if (!cluster.isWorker) {
    throw new Error('startWorker must be run in worker process');
  }

  const workerId = process.env.WORKER_ID ?? String(process.pid);
  const server = app.listen(PORT, () => {
    logger.info({ workerId, port: PORT }, 'Worker listening');
  });

  // マスターからのメッセージを処理
  process.on('message', async (message: any) => {
    if (message.type === 'graceful-shutdown') {
      logger.info({ workerId }, 'Worker graceful shutdown requested');

      // 新しいリクエストの受付を停止
      server.close(async () => {
        // 進行中のリクエストが完了するまで待機
        await gracefulShutdownMiddleware.waitForActiveRequests(5000);
        logger.info({ workerId }, 'Worker shutdown complete');
        process.exit(0);
      });
    }
  });

  // ワーカーのメトリクスを定期送信
  setInterval(() => {
    if (process.send) {
      process.send({
        type: 'metrics',
        data: {
          pid: process.pid,
          memory: process.memoryUsage(),
          uptime: process.uptime(),
        },
      });
    }
  }, 30_000);
}

// src/main.ts — エントリーポイント
if (cluster.isPrimary) {
  const master = new ClusterMaster({ workerCount: 4 });
  master.start();
} else {
  const app = createApp();
  startWorker(app);
}
```

---

## まとめ

Claude CodeでNode.jsクラスターモードを設計する：

1. **CLAUDE.md** にMasterはワーカー管理専用・Workerはリクエスト処理・CPU数分のワーカーを起動・共有状態はRedisに（プロセスローカルに保存しない）を明記
2. **ローリングリロード（SIGUSR2）** でゼロダウンタイムデプロイ——新ワーカーをonlineにしてから旧ワーカーをshutdown。`kill -USR2 <master_pid>`または`npm run reload`で全ワーカーを順番に置き換える
3. **グレースフルシャットダウン** で進行中リクエストを保護——`server.close()`で新リクエストを拒否し、アクティブなリクエストが完了するまで待機。k8sのterminationGracePeriodSeconds（30秒）以内に完了させる
4. **ワーカー間通信はMaster経由** ——ワーカーが直接通信できないため、あるワーカーでキャッシュが更新されたら`process.send({type: 'broadcast', data: ...})`→Masterが全ワーカーに転送。共有状態が必要ならRedisを使う

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
