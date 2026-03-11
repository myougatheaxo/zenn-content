---
title: "Claude CodeでWebSocketをスケールする：Redis Pub/SubとSocket.io Adapter"
emoji: "📡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "websocket"]
published: true
---

## はじめに

WebSocketサーバーを複数台に増やすと、別サーバーに接続しているユーザーにメッセージが届かなくなる。Redis Pub/Subで解決する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにWebSocketスケーリングルールを書く

```markdown
## WebSocketスケーリング設計ルール

### 複数サーバー対応（必須）
- Socket.io Redis Adapterを使用（@socket.io/redis-adapter）
- メッセージはRedis Pub/Subを経由して全サーバーに伝播
- セッション情報はRedisに保存（スティッキーセッション禁止）

### 接続管理
- 接続数・ルーム参加数をメトリクスで監視
- 1ユーザー最大5接続まで（重複接続制限）
- 非アクティブ接続は30分でタイムアウト

### 認証
- JWT認証（接続時にhandshakeでトークン確認）
- 無効なトークンは即切断（401）
- トークン有効期限切れでも切断→再接続フローへ
```

---

## WebSocketスケーリングの生成

```
Socket.ioをRedis Adapterで水平スケーリングする設計を生成してください。

要件：
- @socket.io/redis-adapterでメッセージをブロードキャスト
- JWT認証（接続時に検証）
- ルームへの参加・退出
- 接続数メトリクス
- 最大接続数制限

生成ファイル: src/websocket/socketServer.ts
```

---

## 生成されるWebSocketスケーリング設計

```typescript
// src/websocket/socketServer.ts
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import jwt from 'jsonwebtoken';

export async function createSocketServer(httpServer: HttpServer): Promise<Server> {
  const io = new Server(httpServer, {
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
      credentials: true,
    },
    transports: ['websocket', 'polling'],
    pingTimeout: 60000,
    pingInterval: 25000,
  });

  // Redis Adapterで水平スケーリング
  const pubClient = createClient({ url: process.env.REDIS_URL });
  const subClient = pubClient.duplicate();

  await Promise.all([pubClient.connect(), subClient.connect()]);
  io.adapter(createAdapter(pubClient, subClient));

  logger.info('Socket.io Redis adapter initialized');

  // JWT認証ミドルウェア（接続時に実行）
  io.use(async (socket, next) => {
    const token = socket.handshake.auth.token as string;

    if (!token) {
      return next(new Error('Authentication required'));
    }

    try {
      const payload = jwt.verify(token, process.env.JWT_SECRET!) as {
        userId: string;
        tenantId: string;
      };

      socket.data.userId = payload.userId;
      socket.data.tenantId = payload.tenantId;
      next();
    } catch (err) {
      next(new Error('Invalid token'));
    }
  });

  // 接続数制限ミドルウェア
  io.use(async (socket, next) => {
    const userId = socket.data.userId;
    const connectionKey = `ws:connections:${userId}`;

    const connectionCount = await pubClient.sCard(connectionKey);
    if (connectionCount >= 5) {
      return next(new Error('Too many connections'));
    }

    // 接続を追跡
    await pubClient.sAdd(connectionKey, socket.id);
    await pubClient.expire(connectionKey, 3600); // 1時間TTL

    socket.on('disconnect', async () => {
      await pubClient.sRem(connectionKey, socket.id);
    });

    next();
  });

  io.on('connection', (socket) => {
    const { userId, tenantId } = socket.data;

    logger.info({ userId, socketId: socket.id }, 'WebSocket connected');

    // テナント専用ルームに自動参加
    socket.join(`tenant:${tenantId}`);

    // ユーザー専用ルームに参加
    socket.join(`user:${userId}`);

    socket.on('join:room', async (roomId: string) => {
      // ルーム参加権限チェック
      const hasAccess = await checkRoomAccess(userId, roomId);
      if (!hasAccess) {
        socket.emit('error', { code: 'ROOM_ACCESS_DENIED' });
        return;
      }

      socket.join(roomId);
      socket.to(roomId).emit('user:joined', { userId });
    });

    socket.on('message', async (data: { roomId: string; content: string }) => {
      const { roomId, content } = data;

      // メッセージをDBに保存
      const message = await prisma.message.create({
        data: { roomId, senderId: userId, content },
      });

      // ルーム全員に送信（Redis Adapterが全サーバーに伝播）
      io.to(roomId).emit('message:new', {
        id: message.id,
        content: message.content,
        senderId: userId,
        sentAt: message.createdAt,
      });
    });

    socket.on('disconnect', (reason) => {
      logger.info({ userId, socketId: socket.id, reason }, 'WebSocket disconnected');
    });
  });

  return io;
}
```

---

## テナント全体へのブロードキャスト

```typescript
// サーバー側からテナント全員に通知
async function broadcastToTenant(tenantId: string, event: string, data: unknown): Promise<void> {
  io.to(`tenant:${tenantId}`).emit(event, data);
  // Redis Adapterが全サーバーに伝播させる
}

// 特定ユーザーへの通知（別サーバーに接続していても届く）
async function notifyUser(userId: string, event: string, data: unknown): Promise<void> {
  io.to(`user:${userId}`).emit(event, data);
}
```

---

## まとめ

Claude CodeでWebSocketをスケールする：

1. **CLAUDE.md** にRedis Adapter必須・JWT認証・接続数制限を明記
2. **@socket.io/redis-adapter** でメッセージを全サーバーに伝播
3. **テナント/ユーザールーム** で効率的なターゲット配信
4. **Redis Set** で接続数を追跡・制限

---

*WebSocketの設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
