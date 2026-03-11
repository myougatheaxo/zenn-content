---
title: "Claude Codeでリアルタイムコラボレーションを設計する：OT・CRDT・競合解決"
emoji: "👥"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "websocket", "redis"]
published: true
---

## はじめに

Googleドキュメントのような共同編集——複数ユーザーが同時にテキストを編集しても競合しない実装。Operational Transformation（OT）とCRDTをClaude Codeに設計させる。

---

## CLAUDE.mdにリアルタイムコラボ設計ルールを書く

```markdown
## リアルタイムコラボレーション設計ルール

### 競合解決方式
- シンプルなテキスト編集: OT（Operational Transformation）
- 分散環境・オフライン対応: CRDT（Yjs使用）
- カーソル位置共有: Awareness（Yjs）

### インフラ
- WebSocket: Socket.io（sticky sessions）
- 状態同期: Redis Pub/Sub（マルチサーバー）
- 永続化: 10秒デバウンスでDBに保存
- ドキュメント状態: Redisにバイナリキャッシュ（TTL: 1時間）

### セキュリティ
- ドキュメントIDとユーザーの権限チェック（閲覧/編集）
- ルームはドキュメントIDでスコープ
```

---

## リアルタイムコラボの生成

```
リアルタイム共同編集システムを設計してください。

要件：
- Yjs（CRDT）でドキュメント状態管理
- WebSocketでリアルタイム同期
- カーソル位置共有
- オフライン対応（接続復帰時に自動マージ）
- Redis Pub/Subでマルチサーバー対応

生成ファイル: src/collab/
```

---

## 生成されるリアルタイムコラボ実装

```typescript
// src/collab/documentRoom.ts
import * as Y from 'yjs';
import { encodeStateAsUpdate, applyUpdate, encodeAwarenessUpdate } from 'yjs';
import { Awareness } from 'y-protocols/awareness';
import { Server, Socket } from 'socket.io';

// ドキュメントルーム管理（インメモリ + Redis永続化）
class DocumentRoom {
  private ydoc: Y.Doc;
  private awareness: Awareness;
  private connectedUsers = new Map<string, ConnectedUser>();
  private saveDebounce: NodeJS.Timeout | null = null;

  constructor(
    private readonly documentId: string,
    initialState?: Uint8Array
  ) {
    this.ydoc = new Y.Doc();
    this.awareness = new Awareness(this.ydoc);

    if (initialState) {
      Y.applyUpdate(this.ydoc, initialState);
    }

    // ドキュメント変更を監視して遅延保存
    this.ydoc.on('update', (update: Uint8Array) => {
      this.broadcastUpdate(update);
      this.scheduleSave();
    });

    this.awareness.on('change', (changes: any) => {
      this.broadcastAwareness(encodeAwarenessUpdate(this.awareness, [
        ...changes.added, ...changes.updated, ...changes.removed,
      ]));
    });
  }

  // ユーザー接続
  async addUser(socket: Socket, userId: string, displayName: string): Promise<void> {
    this.connectedUsers.set(socket.id, { userId, displayName, socketId: socket.id });

    // 現在のドキュメント状態を新規接続に送信
    const currentState = Y.encodeStateAsUpdate(this.ydoc);
    socket.emit('doc:state', currentState);

    // Awarenessの現在状態も送信
    const awarenessState = encodeAwarenessUpdate(this.awareness, Array.from(this.awareness.getStates().keys()));
    socket.emit('awareness:update', awarenessState);

    // Pub/Subでマルチサーバーに接続通知
    await redis.publish(`doc:${this.documentId}:join`, JSON.stringify({
      userId, displayName, socketId: socket.id,
    }));
  }

  // クライアントからの更新を処理
  applyClientUpdate(update: Uint8Array, socketId: string): void {
    try {
      // CRDTの競合を自動解決（YJSのベクタークロック）
      Y.applyUpdate(this.ydoc, update);
      // updateイベントが発火してbroadcastUpdateが呼ばれる
    } catch (err) {
      logger.error({ documentId: this.documentId, socketId }, 'Failed to apply update');
    }
  }

  // 全接続に更新をブロードキャスト（送信者を除く）
  private broadcastUpdate(update: Uint8Array): void {
    io.to(`doc:${this.documentId}`).emit('doc:update', update);
  }

  private broadcastAwareness(update: Uint8Array): void {
    io.to(`doc:${this.documentId}`).emit('awareness:update', update);
  }

  // 10秒デバウンスでDBに保存
  private scheduleSave(): void {
    if (this.saveDebounce) clearTimeout(this.saveDebounce);
    this.saveDebounce = setTimeout(() => this.persistDocument(), 10_000);
  }

  private async persistDocument(): Promise<void> {
    const state = Y.encodeStateAsUpdate(this.ydoc);
    const content = this.ydoc.getText('content').toString();

    await prisma.document.update({
      where: { id: this.documentId },
      data: {
        content,
        yjsState: Buffer.from(state),
        lastSavedAt: new Date(),
      },
    });

    // Redisキャッシュも更新
    await redis.set(`doc:state:${this.documentId}`, Buffer.from(state), { EX: 3600 });
  }

  disconnect(socketId: string): void {
    this.connectedUsers.delete(socketId);
    this.awareness.setLocalState(null); // カーソル削除
  }
}
```

```typescript
// src/collab/server.ts — WebSocketハンドラー

const documentRooms = new Map<string, DocumentRoom>();

async function getOrCreateRoom(documentId: string): Promise<DocumentRoom> {
  if (documentRooms.has(documentId)) {
    return documentRooms.get(documentId)!;
  }

  // DBからドキュメント状態を復元
  const doc = await prisma.document.findUnique({ where: { id: documentId } });
  const cachedState = await redis.getBuffer(`doc:state:${documentId}`);

  const state = cachedState
    ? new Uint8Array(cachedState)
    : doc?.yjsState
    ? new Uint8Array(doc.yjsState)
    : undefined;

  const room = new DocumentRoom(documentId, state);
  documentRooms.set(documentId, room);
  return room;
}

io.on('connection', async (socket: Socket) => {
  const { documentId, userId } = socket.handshake.auth;

  // 権限チェック
  const permission = await checkDocumentPermission(userId, documentId);
  if (!permission) {
    socket.emit('error', { code: 'FORBIDDEN' });
    socket.disconnect();
    return;
  }

  const room = await getOrCreateRoom(documentId);
  const user = await prisma.user.findUniqueOrThrow({ where: { id: userId } });

  socket.join(`doc:${documentId}`);
  await room.addUser(socket, userId, user.name);

  // クライアントからのドキュメント更新
  socket.on('doc:update', (update: ArrayBuffer) => {
    if (permission === 'view') return; // 閲覧権限は書き込み不可
    room.applyClientUpdate(new Uint8Array(update), socket.id);
  });

  // カーソル・選択範囲の共有（Awareness）
  socket.on('awareness:update', (update: ArrayBuffer) => {
    room.applyAwarenessUpdate(new Uint8Array(update), socket.id);
  });

  socket.on('disconnect', () => {
    room.disconnect(socket.id);
  });
});
```

---

## フロントエンド統合（React）

```typescript
// src/components/CollaborativeEditor.tsx
import * as Y from 'yjs';
import { useEffect, useRef } from 'react';
import { io } from 'socket.io-client';
import { WebsocketProvider } from 'y-websocket';
import { QuillBinding } from 'y-quill';
import Quill from 'quill';

export function CollaborativeEditor({ documentId }: { documentId: string }) {
  const editorRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const ydoc = new Y.Doc();
    const ytext = ydoc.getText('content');

    // WebSocketプロバイダー（自動再接続・オフライン対応）
    const provider = new WebsocketProvider(
      process.env.NEXT_PUBLIC_WS_URL!,
      documentId,
      ydoc,
      { connect: true }
    );

    // Quillエディタとのバインディング
    const quill = new Quill(editorRef.current!, { theme: 'snow' });
    const binding = new QuillBinding(ytext, quill, provider.awareness);

    // 自分のカーソル情報をAwarenessに設定
    provider.awareness.setLocalStateField('user', {
      name: currentUser.name,
      color: `#${Math.floor(Math.random() * 0xFFFFFF).toString(16)}`,
    });

    return () => {
      binding.destroy();
      provider.destroy();
      ydoc.destroy();
    };
  }, [documentId]);

  return <div ref={editorRef} />;
}
```

---

## まとめ

Claude Codeでリアルタイムコラボレーションを設計する：

1. **CLAUDE.md** にYjsCRDT・WebSocket・Redis Pub/Sub・10秒デバウンス保存を明記
2. **CRDT（Yjs）** でベクタークロックによる競合自動解決——OTより分散環境に強い
3. **Awareness** でカーソル位置・選択範囲を低レイテンシで共有
4. **10秒デバウンス保存** で全キーストロークをDBに書かず負荷を抑制

---

*コラボレーション設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
