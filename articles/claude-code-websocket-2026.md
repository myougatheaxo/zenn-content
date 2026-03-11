---
title: "Claude CodeでWebSocket実装を設計する：リアルタイム通信の標準パターン"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "websocket", "nodejs", "typescript", "リアルタイム"]
published: true
---

## はじめに

WebSocketの実装は接続管理・再接続・ルーム管理など考慮事項が多い。Claude Codeに設計パターンを伝えてコードを生成させると、ベストプラクティスに沿った実装が得られる。

---

## CLAUDE.mdにWebSocketルールを書く

```markdown
## WebSocketルール

### サーバーサイド
- ライブラリ: socket.io（名前空間・ルーム・再接続を標準サポート）
- 全接続はJWT認証を要求（接続時のhandshake）
- ルーム命名規則: `{type}:{id}` 例: `room:uuid`, `user:userId`
- イベント名はsnake_case: `message_sent`, `user_joined`

### クライアントサイド
- 再接続設定: 最大5回、指数バックオフ
- 接続失敗時はUIにフィードバック表示
- イベントリスナーは必ずクリーンアップ（コンポーネントアンマウント時）

### セキュリティ
- メッセージサイズ制限: 64KB
- レート制限: 1ユーザーあたり60メッセージ/分
- XSSを防ぐためメッセージはエスケープして送信

### イベント設計
- クライアント→サーバー: 動詞+名詞 (`send_message`, `join_room`)
- サーバー→クライアント: 名詞+動詞/形容詞 (`message_received`, `user_connected`)
```

---

## Socket.ioサーバーの生成

```
socket.ioを使ったWebSocketサーバーを生成してください。

要件：
- JWT認証（接続時にhandshakeのauthトークンを検証）
- チャットルーム機能（join/leave/message）
- 接続管理（オンラインユーザー一覧をRedisで管理）
- レート制限（60メッセージ/分/ユーザー）
- TypeScript

生成するファイル:
- src/websocket/server.ts（Socket.ioサーバー初期化）
- src/websocket/handlers/chatHandler.ts（チャットイベント）
- src/websocket/middleware/authMiddleware.ts（JWT検証）
- src/websocket/services/presenceService.ts（オンライン管理）
```

---

## 生成されるサーバーの構造例

```typescript
// src/websocket/server.ts
import { Server } from 'socket.io';
import { verifyToken } from './middleware/authMiddleware';
import { registerChatHandlers } from './handlers/chatHandler';

export function initWebSocket(httpServer: HttpServer): Server {
  const io = new Server(httpServer, {
    cors: { origin: process.env.CLIENT_URL, credentials: true },
    maxHttpBufferSize: 64 * 1024, // 64KB
  });

  // JWT認証ミドルウェア
  io.use(verifyToken);

  io.on('connection', (socket) => {
    console.log(`[WS] Connected: ${socket.data.userId}`);

    registerChatHandlers(io, socket);

    socket.on('disconnect', (reason) => {
      console.log(`[WS] Disconnected: ${socket.data.userId}, reason: ${reason}`);
    });
  });

  return io;
}
```

```typescript
// src/websocket/handlers/chatHandler.ts
import { rateLimit } from '../middleware/rateLimitMiddleware';

export function registerChatHandlers(io: Server, socket: Socket) {
  socket.on('join_room', async ({ roomId }) => {
    socket.join(`room:${roomId}`);
    socket.to(`room:${roomId}`).emit('user_joined', {
      userId: socket.data.userId,
      timestamp: Date.now(),
    });
  });

  socket.on('send_message', rateLimit(60, async ({ roomId, content }) => {
    const sanitized = escapeHtml(content);
    if (sanitized.length > 1000) return; // メッセージ長制限

    io.to(`room:${roomId}`).emit('message_received', {
      userId: socket.data.userId,
      content: sanitized,
      timestamp: Date.now(),
    });
  }));
}
```

---

## クライアント側の再接続処理

```
socket.ioクライアントの再接続処理を生成してください（React + TypeScript）。

要件：
- 再接続: 最大5回、指数バックオフ（1s, 2s, 4s, 8s, 16s）
- 接続状態をReact状態として管理（connecting/connected/disconnected/failed）
- 再接続中はUIにスピナー表示
- 5回失敗後は「再読み込みしてください」を表示
- useEffectのクリーンアップでsocket.disconnect()

保存先: src/hooks/useWebSocket.ts
```

```typescript
// 生成されるhookの例
export function useWebSocket(roomId: string) {
  const [status, setStatus] = useState<ConnectionStatus>('connecting');
  const socketRef = useRef<Socket | null>(null);

  useEffect(() => {
    const socket = io(process.env.REACT_APP_WS_URL!, {
      auth: { token: getAuthToken() },
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 16000,
    });

    socket.on('connect', () => setStatus('connected'));
    socket.on('disconnect', () => setStatus('disconnected'));
    socket.on('reconnect_failed', () => setStatus('failed'));

    socketRef.current = socket;

    return () => {
      socket.disconnect();
    };
  }, [roomId]);

  return { socket: socketRef.current, status };
}
```

---

## WebSocketイベントのテスト

```
socket.ioサーバーのユニットテストを生成してください。

テスト対象:
- 有効なJWTで接続できる
- 無効なJWTは接続拒否される
- join_roomでルームに参加できる
- send_messageが他のルームメンバーに届く
- レート制限を超えるとメッセージが無視される

ツール: vitest + socket.io-client（テスト用）
保存先: src/websocket/__tests__/chatHandler.test.ts
```

---

## まとめ

Claude CodeでWebSocket実装を設計する：

1. **CLAUDE.md** に接続管理・セキュリティ・イベント命名規則を明記
2. **socket.io** でルーム・再接続・認証を標準的に実装
3. **クライアントhook** で再接続・接続状態管理を統一
4. **テスト** で接続フローと認証を自動検証

---

*WebSocketとリアルタイム通信のセキュリティレビューは **Security Pack（¥1,480）** の `/security-check` で対応。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
