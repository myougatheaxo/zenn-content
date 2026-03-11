---
title: "Claude Codeで読み後整合性（Read-After-Write）を設計する：DB読み取りレプリカ・モノトニック読み取り"
emoji: "📖"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "redis"]
published: true
published_at: "2026-03-15 22:00"
---

## はじめに

「投稿したのに一覧に出てこない」——読み取りレプリカの遅延が原因。Read-After-Write整合性（自分が書いたものは自分が読める）とモノトニック読み取り（時間が戻らない）を保証する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにRead-After-Write設計ルールを書く

```markdown
## 読み後整合性設計ルール

### 基本方針
- Read-After-Write: ユーザーが書き込んだ直後のクエリはプライマリDBで実行
- モノトニック読み取り: 同一セッション内でデータが「過去に戻らない」
- スティッキーレプリカ: セッション内は同じレプリカに貼り付く

### セッション管理
- 書き込み後15秒以内のリクエスト: プライマリで読み取り
- 15秒経過後: レプリカでの読み取りに移行
- Redisにwrite_timestampを保存（TTL: 15秒）

### レプリカ選択ルール
- レプリカのlast_received_lsnをチェック（ポーリング不可→セッションCookieで代替）
- クリティカルなエンドポイント（ユーザープロフィール編集後確認等）は常にプライマリ
```

---

## Read-After-Write実装の生成

```
Read-After-Write整合性を保証するDB接続管理を設計してください。

要件：
- 書き込み後の一時的なプライマリ読み取り
- セッションスティッキーネス
- レプリカラグの検出と回避
- ミドルウェアでの自動切り替え

生成ファイル: src/db/readAfterWrite/
```

---

## 生成されるRead-After-Write実装

```typescript
// src/db/readAfterWrite/dbRouter.ts — DBルーター

import { PrismaClient } from '@prisma/client';

// プライマリDBとレプリカのクライアント
export const primaryDB = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_URL_PRIMARY } },
});

export const replicaDB = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_URL_REPLICA } },
});

const READ_AFTER_WRITE_TTL_MS = 15_000; // 書き込み後15秒間はプライマリで読む

export class DBRouter {
  // 書き込みが発生したことを記録
  async markWrite(userId: string): Promise<void> {
    await redis.set(`raw:${userId}`, Date.now().toString(), { PX: READ_AFTER_WRITE_TTL_MS });
  }

  // ユーザーのread後整合性が必要かチェック
  async needsPrimary(userId: string | undefined): Promise<boolean> {
    if (!userId) return false;
    const writeTime = await redis.get(`raw:${userId}`);
    return writeTime !== null; // TTL内ならプライマリを使う
  }

  // 読み取りクライアントを選択
  async getReadClient(userId: string | undefined): Promise<PrismaClient> {
    const usePrimary = await this.needsPrimary(userId);
    return usePrimary ? primaryDB : replicaDB;
  }
}

export const dbRouter = new DBRouter();
```

```typescript
// src/db/readAfterWrite/middleware.ts — Expressミドルウェア

// 書き込み操作の後にDBルーターへ通知
export function trackWrite(req: Request, res: Response, next: NextFunction) {
  const originalJson = res.json.bind(res);

  res.json = (body) => {
    // 書き込み成功レスポンス（201, 200 for mutations）の場合にマーク
    if (req.method !== 'GET' && res.statusCode < 400 && req.userId) {
      // 非同期でマーク（レスポンスをブロックしない）
      dbRouter.markWrite(req.userId).catch(err => logger.warn(err, 'Failed to mark write'));
    }
    return originalJson(body);
  };

  next();
}

// 読み取りクライアントを自動選択
export function attachReadDB(req: Request, res: Response, next: NextFunction) {
  if (req.method === 'GET') {
    // req.dbにユーザーに適したクライアントをセット（非同期）
    dbRouter.getReadClient(req.userId).then(client => {
      req.db = client;
      next();
    }).catch(next);
  } else {
    // 書き込みは常にプライマリ
    req.db = primaryDB;
    next();
  }
}
```

```typescript
// src/db/readAfterWrite/monotonicRead.ts — モノトニック読み取り

// セッション内での読み取りタイムスタンプを記録
export class MonotonicReadTracker {
  // 読み取り時のサーバータイムスタンプを記録
  async recordRead(sessionId: string): Promise<void> {
    const existing = await redis.get(`monotonic:${sessionId}`);
    const current = Date.now();
    // 既存より新しいタイムスタンプのみ更新（モノトニック増加）
    if (!existing || current > parseInt(existing)) {
      await redis.set(`monotonic:${sessionId}`, current.toString(), { EX: 1800 }); // 30分
    }
  }

  async getLastReadTime(sessionId: string): Promise<number> {
    const ts = await redis.get(`monotonic:${sessionId}`);
    return ts ? parseInt(ts) : 0;
  }

  // レプリカのラグを確認し、モノトニック読み取りを保証できるかチェック
  async canUseReplica(sessionId: string): Promise<boolean> {
    const lastRead = await this.getLastReadTime(sessionId);
    if (lastRead === 0) return true;

    // レプリカのWALラグを確認（pg_stat_replication経由）
    const replicaLag = await getReplicaLagMs();

    // レプリカのデータが最後に読んだ時刻より新しければOK
    const replicaDataAge = Date.now() - replicaLag;
    return replicaDataAge >= lastRead;
  }
}

// PostgreSQLレプリカラグ取得
async function getReplicaLagMs(): Promise<number> {
  const cacheKey = 'replica:lag_ms';
  const cached = await redis.get(cacheKey);
  if (cached) return parseInt(cached);

  const result = await primaryDB.$queryRaw<Array<{ lag_ms: number }>>`
    SELECT EXTRACT(EPOCH FROM replay_lag) * 1000 AS lag_ms
    FROM pg_stat_replication
    ORDER BY lag_ms DESC
    LIMIT 1
  `;

  const lagMs = result[0]?.lag_ms ?? 0;
  await redis.set(cacheKey, lagMs.toString(), { EX: 5 }); // 5秒キャッシュ
  return lagMs;
}
```

```typescript
// 完全な使用例: 投稿作成→一覧取得

// POST /api/posts — 書き込み
router.post('/api/posts', requireAuth, trackWrite, async (req, res) => {
  const post = await primaryDB.post.create({
    data: { title: req.body.title, content: req.body.content, userId: req.userId },
  });

  // write後のフラグを設定（middlewareのtrackWriteが自動的にmarkWrite呼び出し）
  res.status(201).json(post);
});

// GET /api/posts — 一覧取得（Read-After-Write保証）
router.get('/api/posts', requireAuth, attachReadDB, async (req, res) => {
  const tracker = new MonotonicReadTracker();
  const sessionId = req.session?.id ?? req.userId;

  // モノトニック読み取り保証
  let db = req.db; // attachReadDBが設定済み
  const canUseReplica = await tracker.canUseReplica(sessionId);

  if (!canUseReplica) {
    db = primaryDB; // レプリカラグが大きい場合はプライマリに切り替え
    logger.debug({ sessionId }, 'Switched to primary: replica lag too high');
  }

  const posts = await db.post.findMany({
    where: { userId: req.userId, deletedAt: null },
    orderBy: { createdAt: 'desc' },
    take: 20,
  });

  await tracker.recordRead(sessionId);
  res.json(posts);
});
```

---

## まとめ

Claude CodeでRead-After-Write整合性を設計する：

1. **CLAUDE.md** に書き込み後15秒はプライマリ読み取り・モノトニック読み取り保証・Redisでwrite_timestamp管理を明記
2. **trackWriteミドルウェア** がレスポンス成功時に自動でRedisフラグを設定——アプリコードに整合性ロジックを書かなくてよい
3. **attachReadDB** がユーザーのwrite履歴を見てプライマリ/レプリカを自動選択——TTL15秒で自然にレプリカへ移行
4. **pg_stat_replication** でレプリカラグをリアルタイム監視——ラグが大きい場合はプライマリへ自動フォールバック

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
