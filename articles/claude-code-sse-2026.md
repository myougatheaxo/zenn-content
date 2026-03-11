---
title: "Claude CodeでServer-Sent Eventsを実装する：WebSocketより簡単なリアルタイム通信"
emoji: "📡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "sse", "リアルタイム"]
published: true
---

## はじめに

サーバーからクライアントへの一方向のリアルタイム配信——通知・進捗表示・ライブフィードにはWebSocketより実装が簡単なSSE（Server-Sent Events）が適している。Claude Codeに設計させる。

---

## CLAUDE.mdにSSEルールを書く

```markdown
## Server-Sent Events（SSE）ルール

### 使用場面（SSEが適切）
- サーバー → クライアントの一方向配信
- 通知・進捗表示・ライブフィード・ダッシュボード更新
- クライアント → サーバーの通信が不要な場合

### WebSocketを使う場面（SSEではなく）
- 双方向通信（チャット・ゲーム・コラボレーション）
- バイナリデータの送受信

### 実装ルール
- Content-Type: text/event-stream
- Cache-Control: no-cache
- Connection: keep-alive
- X-Accel-Buffering: no（Nginxのバッファリング無効化）
- 認証: クエリパラメータでトークン（Authorizationヘッダーは使えない）
- 再接続: Last-Event-IDヘッダーを使って途中から再開
- ハートビート: 30秒ごとにコメントを送信（接続維持）

### セキュリティ
- SSEエンドポイントも認証必須
- CORSを適切に設定
- 1ユーザーあたり最大接続数を制限（5接続）
```

---

## SSEサーバーの生成

```
Express + TypeScriptでSSEエンドポイントを生成してください。

要件：
- エンドポイント: GET /api/events
- JWT認証（クエリパラメータ: ?token=xxx）
- イベントタイプ: notification, update, heartbeat
- ハートビート: 30秒ごと
- 接続管理: userId別に接続をMapで管理
- 接続切断時の自動クリーンアップ

生成するファイル:
- src/sse/sseManager.ts（接続管理）
- src/routes/events.ts（SSEエンドポイント）
```

---

## 生成されるSSEサーバー

```typescript
// src/sse/sseManager.ts
import { Response } from 'express';

interface SSEClient {
  userId: string;
  res: Response;
  lastEventId?: string;
}

class SSEManager {
  private clients = new Map<string, Set<SSEClient>>();
  private heartbeatInterval: NodeJS.Timeout;

  constructor() {
    // 30秒ごとにハートビートを送信
    this.heartbeatInterval = setInterval(() => {
      this.broadcast({ type: 'heartbeat', data: {} });
    }, 30_000);
  }

  addClient(userId: string, client: SSEClient): void {
    if (!this.clients.has(userId)) {
      this.clients.set(userId, new Set());
    }

    const userClients = this.clients.get(userId)!;

    // 最大5接続制限
    if (userClients.size >= 5) {
      client.res.end();
      return;
    }

    userClients.add(client);

    client.res.on('close', () => {
      userClients.delete(client);
      if (userClients.size === 0) {
        this.clients.delete(userId);
      }
    });
  }

  sendToUser(userId: string, event: SSEEvent): void {
    const userClients = this.clients.get(userId);
    if (!userClients) return;

    userClients.forEach((client) => {
      this.sendEvent(client.res, event);
    });
  }

  broadcast(event: SSEEvent): void {
    this.clients.forEach((clients) => {
      clients.forEach((client) => {
        this.sendEvent(client.res, event);
      });
    });
  }

  private sendEvent(res: Response, event: SSEEvent): void {
    if (event.id) res.write(`id: ${event.id}\n`);
    res.write(`event: ${event.type}\n`);
    res.write(`data: ${JSON.stringify(event.data)}\n\n`);
  }
}

export const sseManager = new SSEManager();
```

```typescript
// src/routes/events.ts
import { Router } from 'express';
import { verifyToken } from '../middleware/auth';
import { sseManager } from '../sse/sseManager';

export const eventsRouter = Router();

eventsRouter.get('/events', async (req, res) => {
  // クエリパラメータでJWT認証（EventSourceはAuthorizationヘッダーを使えない）
  const token = req.query.token as string;
  const user = await verifyToken(token);
  if (!user) return res.status(401).end();

  // SSEヘッダーを設定
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no'); // Nginx用
  res.flushHeaders();

  // 接続を登録
  sseManager.addClient(user.id, {
    userId: user.id,
    res,
    lastEventId: req.headers['last-event-id'] as string,
  });

  // 接続確認イベント
  res.write(`event: connected\ndata: {"userId": "${user.id}"}\n\n`);
});
```

---

## クライアント側の実装

```
SSEのクライアント側をReact + TypeScriptで実装してください。

要件：
- カスタムhook: useSSE(url)
- イベント種別ごとにコールバック
- 自動再接続（Last-Event-IDを使って途中から再開）
- 接続失敗時はUIにエラー表示
- コンポーネントアンマウント時にEventSourceをclose

保存先: src/hooks/useSSE.ts
```

```typescript
// src/hooks/useSSE.ts
import { useEffect, useRef, useCallback } from 'react';

interface SSEOptions {
  onNotification?: (data: unknown) => void;
  onUpdate?: (data: unknown) => void;
  onError?: (error: Event) => void;
}

export function useSSE(url: string, options: SSEOptions) {
  const eventSourceRef = useRef<EventSource | null>(null);

  const connect = useCallback(() => {
    const es = new EventSource(url);

    es.addEventListener('notification', (e) => {
      options.onNotification?.(JSON.parse(e.data));
    });

    es.addEventListener('update', (e) => {
      options.onUpdate?.(JSON.parse(e.data));
    });

    es.onerror = (e) => {
      options.onError?.(e);
      // EventSourceは自動再接続する（Last-Event-IDを送信）
    };

    eventSourceRef.current = es;
  }, [url]);

  useEffect(() => {
    connect();
    return () => {
      eventSourceRef.current?.close();
    };
  }, [connect]);
}
```

---

## まとめ

Claude CodeでSSEを実装する：

1. **CLAUDE.md** にSSEの使用場面・ヘッダー設定・認証ルールを明記
2. **SSEManager** で接続をuserIdベースで管理
3. **ハートビート** で長時間接続を維持
4. **クライアントhook** で自動再接続を実装

---

*リアルタイム通信のセキュリティレビューは **Security Pack（¥1,480）** の `/security-check` で自動化できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
