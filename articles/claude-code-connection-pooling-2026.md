---
title: "Claude Codeで接続プールを設計する：PrismaとPostgreSQLの最適なコネクション管理"
emoji: "🏊"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "prisma"]
published: true
---

## はじめに

Node.jsサーバーが複数プロセス起動すると、それぞれがDB接続プールを持つ——100プロセス×10接続=1000接続でPostgreSQLが限界を超える。Claude Codeに接続プール設計を生成させる。

---

## CLAUDE.mdに接続プールルールを書く

```markdown
## 接続プール設計ルール

### プールサイズ（必須）
- Prismaのconnection_limit: CPU数*2+1（デフォルトの10は変更）
- PostgreSQLのmax_connections: サーバースペックに応じて設定
- 複数プロセス環境: Prisma Accelerate or PgBouncer でプール集約

### Singleton
- PrismaClientは1インスタンスのみ（new PrismaClient()を複数作成禁止）
- 開発時のホットリロードで重複作成されないよう globalThis を使う

### 接続エラー
- 接続失敗時はリトライ（指数バックオフ）
- 全プールが枯渇したらキューで待機（タイムアウト30秒）
- 接続エラーはSlack通知
```

---

## 接続プール設計の生成

```
PostgreSQL接続プールを最適化してください。

環境:
- Node.js (複数ワーカー × 4プロセス)
- PostgreSQL (max_connections=100)
- Prisma ORM

要件：
- PrismaClientシングルトン
- 開発ホットリロード対応
- 接続数の計算式
- タイムアウト設定

生成ファイル: src/lib/prisma.ts
```

---

## 生成される接続プール設計

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client';

// 接続数の計算
// PostgreSQL max_connections = 100
// 予約: 10 (PostgreSQL管理・監視用)
// 利用可能: 90
// プロセス数: 4 (pm2 cluster mode)
// 1プロセスあたりの上限: 90 / 4 = 22
// connection_limit = 20 (余裕を持たせる)
const CONNECTION_LIMIT = Number(process.env.DB_POOL_SIZE ?? 20);

function createPrismaClient() {
  const databaseUrl = new URL(process.env.DATABASE_URL!);
  databaseUrl.searchParams.set('connection_limit', String(CONNECTION_LIMIT));
  databaseUrl.searchParams.set('pool_timeout', '30'); // 30秒でタイムアウト

  return new PrismaClient({
    datasources: {
      db: { url: databaseUrl.toString() },
    },
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'warn', 'error']
      : ['warn', 'error'],
  });
}

// シングルトンパターン（開発ホットリロード対応）
declare global {
  // eslint-disable-next-line no-var
  var __prismaClient: PrismaClient | undefined;
}

export const prisma =
  globalThis.__prismaClient ??
  (globalThis.__prismaClient = createPrismaClient());

// 開発環境以外でglobalにキャッシュしない
if (process.env.NODE_ENV !== 'development') {
  globalThis.__prismaClient = undefined;
}
```

---

## 接続数の監視

```typescript
// src/lib/dbMonitor.ts
// Prismaのメトリクスで接続数を監視
const prismaWithMetrics = new PrismaClient({
  // Prisma 5.x でのメトリクス
});

export async function getDatabaseMetrics() {
  // $metrics.json() は Prisma 5.x から利用可能
  const metrics = await (prisma as any).$metrics.json();
  return {
    activeConnections: metrics.gauges.find((g: any) => g.key === 'prisma_pool_connections_open')?.value ?? 0,
    idleConnections: metrics.gauges.find((g: any) => g.key === 'prisma_pool_connections_idle')?.value ?? 0,
    waitQueueSize: metrics.gauges.find((g: any) => g.key === 'prisma_pool_connections_busy')?.value ?? 0,
  };
}

// 定期的に接続数を監視してアラート
setInterval(async () => {
  const metrics = await getDatabaseMetrics();

  if (metrics.activeConnections > CONNECTION_LIMIT * 0.8) {
    logger.warn({ metrics }, 'DB connection pool near capacity');
    await notifySlack(`⚠️ DB pool at ${Math.round(metrics.activeConnections / CONNECTION_LIMIT * 100)}% capacity`);
  }
}, 60_000); // 1分毎
```

---

## PgBouncer（複数サービス環境）

```ini
# pgbouncer.ini
# 複数のアプリサーバーからの接続をまとめて管理

[databases]
myapp = host=postgres port=5432 dbname=myapp

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0

# トランザクション単位でプールを共有（最も効率的）
pool_mode = transaction
max_client_conn = 1000   # クライアント（アプリ）の最大接続数
default_pool_size = 20   # PostgreSQLへの実際の接続数

# 接続タイムアウト
server_connect_timeout = 5
client_idle_timeout = 60
```

---

## まとめ

Claude Codeで接続プールを設計する：

1. **CLAUDE.md** にconnection_limit計算式・シングルトン必須・複数プロセス対策を明記
2. **PrismaClientシングルトン** でglobalThis使用（ホットリロード対応）
3. **connection_limit** を環境変数で設定（プロセス数から逆算）
4. **PgBouncer** で複数サーバー間のコネクションを集約

---

*接続プール設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
