---
title: "Claude CodeでセッションとRedis管理を設計する：スティッキーセッション・分散セッション・期限管理"
emoji: "🍪"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "security"]
published: true
---

## はじめに

複数サーバーにまたがるセッション管理——Redisに集中管理してどのサーバーにリクエストが届いても同じセッションを参照できる分散セッションをClaude Codeに設計させる。

---

## CLAUDE.mdにセッション管理設計ルールを書く

```markdown
## セッション管理設計ルール

### セッションストレージ
- Redis: 全サーバー共有（MemoryStoreは単一サーバーのみ→本番禁止）
- セッションID: crypto.randomBytes(32)以上のランダム値
- Prefix: "sess:" でRedisキーを統一

### セキュリティ
- Cookie: httpOnly=true, secure=true, sameSite='strict'
- セッション固定攻撃防止: ログイン時にIDを再生成（regenerate）
- 有効期限: アイドル30分 + 絶対上限24時間

### パフォーマンス
- セッションデータは軽量に（ユーザーID・ロールのみ）
- 詳細データはDBから取得（セッションに詰め込まない）
- セッションの書き込みは変更がある場合のみ（saveUninitialized: false）
```

---

## セッション管理システムの生成

```
Redisを使った分散セッション管理を設計してください。

要件：
- Expressセッション（express-session）
- Redisストア（connect-redis）
- セッション固定攻撃防止
- スライディングウィンドウ期限更新
- 強制ログアウト（全デバイス）

生成ファイル: src/session/
```

---

## 生成されるセッション管理実装

```typescript
// src/session/config.ts
import session from 'express-session';
import { RedisStore } from 'connect-redis';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

// Redisセッションストア設定
const store = new RedisStore({
  client: redis,
  prefix: 'sess:',        // すべてのキーに"sess:"プレフィックス
  ttl: 30 * 60,           // TTL: 30分（アクセスごとにリセット）
  disableTouch: false,    // touch()で期限を延長する
});

export const sessionMiddleware = session({
  store,
  secret: process.env.SESSION_SECRET!,  // 32バイト以上のランダム値
  resave: false,           // 変更がなければ保存しない
  saveUninitialized: false, // 未認証セッションを保存しない
  rolling: true,           // リクエストごとにCookieの期限を延長
  cookie: {
    httpOnly: true,        // JavaScriptからアクセス不可
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',    // CSRF防止
    maxAge: 24 * 60 * 60 * 1000, // Cookie最大24時間
  },
  genid: () => crypto.randomUUID(), // セッションID生成
});
```

```typescript
// src/session/authSession.ts

// セッションの型定義
declare module 'express-session' {
  interface SessionData {
    userId: string;
    role: string;
    loginAt: number;
    absoluteExpiry: number; // 絶対有効期限
    deviceId: string;
  }
}

const ABSOLUTE_SESSION_HOURS = 24; // 最大24時間でログアウト強制

// ログイン時: セッション固定攻撃を防ぐためIDを再生成
export async function createUserSession(
  req: Request,
  userId: string,
  role: string
): Promise<void> {
  // セッション固定攻撃防止: 既存セッションIDを廃棄して新規発行
  await new Promise<void>((resolve, reject) => {
    req.session.regenerate((err) => {
      if (err) reject(err);
      else resolve();
    });
  });

  const now = Date.now();
  req.session.userId = userId;
  req.session.role = role;
  req.session.loginAt = now;
  req.session.absoluteExpiry = now + ABSOLUTE_SESSION_HOURS * 60 * 60 * 1000;
  req.session.deviceId = crypto.randomUUID();

  // アクティブセッション一覧をRedisのSetで追跡
  await redis.sadd(`user:sessions:${userId}`, req.session.id);

  await new Promise<void>((resolve, reject) => {
    req.session.save((err) => {
      if (err) reject(err);
      else resolve();
    });
  });

  logger.info({ userId, sessionId: req.session.id }, 'Session created');
}

// セッション検証ミドルウェア
export function requireSession(req: Request, res: Response, next: NextFunction) {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  // 絶対有効期限チェック（rolling設定でも強制ログアウト）
  if (Date.now() > req.session.absoluteExpiry) {
    req.session.destroy(() => {});
    return res.status(401).json({ error: 'Session expired. Please login again.' });
  }

  next();
}

// ログアウト: 単一デバイス
export async function destroySession(req: Request): Promise<void> {
  const userId = req.session.userId;
  const sessionId = req.session.id;

  if (userId) {
    await redis.srem(`user:sessions:${userId}`, sessionId);
  }

  await new Promise<void>((resolve) => {
    req.session.destroy(() => resolve());
  });
}

// 強制ログアウト: 全デバイス（パスワード変更・不正アクセス検知時）
export async function destroyAllSessions(userId: string): Promise<number> {
  // ユーザーのアクティブセッション一覧を取得
  const sessionIds = await redis.smembers(`user:sessions:${userId}`);

  if (sessionIds.length === 0) return 0;

  // 全セッションをRedisから削除
  const pipeline = redis.pipeline();
  for (const sessionId of sessionIds) {
    pipeline.del(`sess:${sessionId}`);
  }
  // セッション一覧も削除
  pipeline.del(`user:sessions:${userId}`);
  await pipeline.exec();

  logger.info({ userId, sessionCount: sessionIds.length }, 'All sessions destroyed');
  return sessionIds.length;
}

// アクティブセッション一覧（デバイス管理UI用）
export async function getActiveSessions(userId: string): Promise<ActiveSession[]> {
  const sessionIds = await redis.smembers(`user:sessions:${userId}`);

  const sessions = await Promise.all(
    sessionIds.map(async (id) => {
      const raw = await redis.get(`sess:${id}`);
      if (!raw) return null;

      const data = JSON.parse(raw);
      return {
        sessionId: id,
        loginAt: new Date(data.loginAt),
        deviceId: data.deviceId,
        expiresAt: new Date(data.absoluteExpiry),
      };
    })
  );

  return sessions.filter(Boolean) as ActiveSession[];
}
```

---

## セッションハイジャック対策

```typescript
// src/session/securityMiddleware.ts

// セッションハイジャック検知: IPアドレスとUser-Agentをバインド
export function sessionBindingMiddleware(req: Request, res: Response, next: NextFunction) {
  if (!req.session.userId) return next();

  const currentIP = req.ip;
  const currentUA = req.headers['user-agent'] ?? '';

  // 初回アクセス時にバインド情報を記録
  if (!req.session.boundIP) {
    req.session.boundIP = currentIP;
    req.session.boundUA = currentUA;
    return next();
  }

  // IPが大きく変わった場合（VPN切り替えは許容、地域が変わった場合は警告）
  const ipChanged = req.session.boundIP !== currentIP;
  const uaChanged = req.session.boundUA !== currentUA;

  if (ipChanged && uaChanged) {
    // IP + UAが両方変わった = セッションハイジャックの可能性が高い
    logger.warn({
      userId: req.session.userId,
      originalIP: req.session.boundIP,
      currentIP,
    }, 'Possible session hijacking detected');

    req.session.destroy(() => {});
    return res.status(401).json({
      error: 'Session invalidated for security reasons. Please login again.',
    });
  }

  next();
}
```

---

## まとめ

Claude CodeでRedisセッション管理を設計する：

1. **CLAUDE.md** にRedisストア必須・Cookie設定・アイドル30分+絶対24時間制限を明記
2. **ログイン時にsession.regenerate()** でセッション固定攻撃を防ぐ
3. **user:sessions:{userId} のRedis Set** で全デバイスのセッションを追跡→一括ログアウト可能
4. **絶対有効期限チェック** でrollingCookieでも24時間で強制ログアウト

---

*セッション管理設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
