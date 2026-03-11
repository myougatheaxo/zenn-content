---
title: "Claude Codeでプッシュ通知を設計する：Firebase FCMとデバイストークン管理"
emoji: "🔔"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "firebase", "bullmq"]
published: true
---

## はじめに

プッシュ通知は「送るだけ」では不十分——無効なトークンを送り続けるとFCMレートが下がる、サイレント通知が欲しい時に通知が出る。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにプッシュ通知ルールを書く

```markdown
## プッシュ通知設計ルール

### トークン管理
- デバイストークンはユーザーごとに複数保存（マルチデバイス対応）
- FCMから「token不正」エラーが返ったら即削除
- トークンの最終使用日時を更新（90日使われないトークンは削除）

### 送信
- 1ユーザーへの送信は全デバイスに対してsendEach（並列）
- 大量送信（1000件超）はBullMQキューで非同期
- FCMのバッチAPI使用（最大500件/リクエスト）

### コンテンツ
- title/bodyは文字数制限内（iOS: 178文字、Android: なし）
- data payload: JSONで追加データを渡す（アプリ内のナビゲーション等）
- サイレント通知（data-only）はバックグラウンド更新に使用
```

---

## プッシュ通知の生成

```
Firebase FCMプッシュ通知システムを設計してください。

要件：
- デバイストークンのDB管理（複数デバイス対応）
- 1ユーザーへの送信（全デバイスに並列送信）
- 無効トークンの自動削除
- 大量送信はBullMQで非同期
- 通知送信履歴を記録

生成ファイル:
- src/services/pushNotificationService.ts
- src/workers/notificationWorker.ts
```

---

## 生成されるプッシュ通知システム

```typescript
// src/services/pushNotificationService.ts
import { initializeApp, cert } from 'firebase-admin/app';
import { getMessaging, MulticastMessage } from 'firebase-admin/messaging';

initializeApp({
  credential: cert({
    projectId: process.env.FCM_PROJECT_ID,
    clientEmail: process.env.FCM_CLIENT_EMAIL,
    privateKey: process.env.FCM_PRIVATE_KEY?.replace(/\\n/g, '\n'),
  }),
});

const messaging = getMessaging();

export interface PushNotificationPayload {
  title: string;
  body: string;
  data?: Record<string, string>; // 文字列のみ
  imageUrl?: string;
  sound?: string;
}

// デバイストークンを登録
export async function registerDeviceToken(
  userId: string,
  token: string,
  platform: 'ios' | 'android'
): Promise<void> {
  await prisma.deviceToken.upsert({
    where: { token },
    create: { userId, token, platform, lastUsedAt: new Date() },
    update: { userId, lastUsedAt: new Date() }, // 既存トークンの最終使用日時を更新
  });
}

// 1ユーザーの全デバイスに送信
export async function sendToUser(
  userId: string,
  payload: PushNotificationPayload
): Promise<{ sent: number; failed: number }> {
  const tokens = await prisma.deviceToken.findMany({
    where: { userId },
    select: { token: true },
  });

  if (tokens.length === 0) return { sent: 0, failed: 0 };

  const tokenList = tokens.map(t => t.token);

  const message: MulticastMessage = {
    tokens: tokenList,
    notification: {
      title: payload.title,
      body: payload.body,
      imageUrl: payload.imageUrl,
    },
    data: payload.data,
    apns: {
      payload: {
        aps: { sound: payload.sound ?? 'default' },
      },
    },
    android: {
      notification: { sound: payload.sound ?? 'default' },
    },
  };

  const result = await messaging.sendEachForMulticast(message);

  // 無効なトークンを削除
  const invalidTokens: string[] = [];
  result.responses.forEach((response, index) => {
    if (!response.success) {
      const errorCode = response.error?.code;
      if (
        errorCode === 'messaging/invalid-registration-token' ||
        errorCode === 'messaging/registration-token-not-registered'
      ) {
        invalidTokens.push(tokenList[index]);
      }
    }
  });

  if (invalidTokens.length > 0) {
    await prisma.deviceToken.deleteMany({
      where: { token: { in: invalidTokens } },
    });
    logger.info({ count: invalidTokens.length }, 'Invalid FCM tokens removed');
  }

  return {
    sent: result.successCount,
    failed: result.failureCount,
  };
}
```

```typescript
// src/services/pushNotificationService.ts (続き)
// 大量送信用: ユーザーリストに送信
export async function queueBulkNotification(params: {
  userIds: string[];
  payload: PushNotificationPayload;
  scheduledAt?: Date;
}): Promise<void> {
  // BullMQキューに追加（1ユーザーずつジョブを作成）
  const jobs = params.userIds.map(userId => ({
    name: 'send-notification',
    data: { userId, payload: params.payload },
    opts: {
      delay: params.scheduledAt ? params.scheduledAt.getTime() - Date.now() : 0,
    },
  }));

  await notificationQueue.addBulk(jobs);
}

// 90日以上使われていないトークンを削除（定期実行）
export async function cleanupStaleTokens(): Promise<number> {
  const cutoff = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000);
  const result = await prisma.deviceToken.deleteMany({
    where: { lastUsedAt: { lt: cutoff } },
  });
  return result.count;
}
```

---

## Prismaスキーマ

```prisma
model DeviceToken {
  id         String   @id @default(cuid())
  userId     String
  token      String   @unique
  platform   String   // 'ios' | 'android'
  lastUsedAt DateTime @default(now())
  createdAt  DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([lastUsedAt])
}
```

---

## まとめ

Claude Codeでプッシュ通知を設計する：

1. **CLAUDE.md** にトークン管理・無効トークン削除・大量送信はキューを明記
2. **sendEachForMulticast** で1ユーザーの全デバイスに並列送信
3. **無効トークン自動削除** でFCMレートの低下を防ぐ
4. **BullMQキュー** で大量送信を非同期処理

---

*プッシュ通知の設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
