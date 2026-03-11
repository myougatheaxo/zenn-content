---
title: "Claude Codeでマルチチャネル通知システムを設計する：Email・Push・Slack・優先度キュー"
emoji: "📢"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "aws"]
published: true
published_at: "2026-03-12 15:00"
---

## はじめに

メール・プッシュ・Slack・SMSを一元管理する通知システム——ユーザーの設定に応じたチャネル選択と優先度キューで確実な通知配信をClaude Codeに設計させる。

---

## CLAUDE.mdに通知システム設計ルールを書く

```markdown
## 通知システム設計ルール

### チャネル優先度
- CRITICAL（障害・セキュリティ）: SMS + Slack + Email全送信
- HIGH（支払い失敗・期限切れ）: Email + Push
- MEDIUM（確認・完了）: Email or Push（ユーザー設定）
- LOW（マーケティング）: Email（同意がある場合のみ）

### 配信ルール
- 重複防止: 同一ユーザー×イベント×24時間以内は1回のみ
- 静寂時間: 22時〜8時はプッシュ通知送信しない（CRITICAL除く）
- レート制限: Email最大50件/時（SESのレート上限対策）
- 失敗リトライ: 最大3回、指数バックオフ

### ユーザー設定
- チャネルごとにオン/オフ設定可能
- 通知タイプごとに設定可能
- 全通知オフも可（CRITICAL除く）
```

---

## 通知システムの生成

```
マルチチャネル通知システムを設計してください。

要件：
- Email（SES）・Push（FCM）・Slack通知
- 優先度キュー（BullMQ）
- 重複防止
- ユーザー設定
- 配信ログ

生成ファイル: src/notifications/
```

---

## 生成される通知システム実装

```typescript
// src/notifications/notificationService.ts

type NotificationPriority = 'critical' | 'high' | 'medium' | 'low';
type NotificationChannel = 'email' | 'push' | 'slack' | 'sms';

interface NotificationRequest {
  userId: string;
  type: string;          // 'payment_failed', 'order_confirmed' など
  priority: NotificationPriority;
  data: Record<string, any>;
  channels?: NotificationChannel[]; // 省略時は優先度とユーザー設定から決定
  deduplicationKey?: string; // 省略時は type:userId:date
}

export class NotificationService {
  private queues: Record<NotificationPriority, Queue>;

  constructor() {
    // 優先度ごとに別キュー（criticalは即時処理）
    this.queues = {
      critical: new Queue('notifications:critical', { connection: redis }),
      high:     new Queue('notifications:high',     { connection: redis }),
      medium:   new Queue('notifications:medium',   { connection: redis }),
      low:      new Queue('notifications:low',      { connection: redis }),
    };
  }

  async send(request: NotificationRequest): Promise<string> {
    // 重複チェック
    const dedupKey = request.deduplicationKey
      ?? `notif:dedup:${request.userId}:${request.type}:${new Date().toISOString().split('T')[0]}`;

    const alreadySent = await redis.get(dedupKey);
    if (alreadySent) {
      logger.info({ userId: request.userId, type: request.type }, 'Notification deduplicated');
      return `deduplicated:${dedupKey}`;
    }

    // ユーザー通知設定を確認
    const userPrefs = await this.getUserPreferences(request.userId);
    const channels = await this.resolveChannels(request, userPrefs);

    if (channels.length === 0) {
      logger.info({ userId: request.userId, type: request.type }, 'All channels disabled by user');
      return 'all_channels_disabled';
    }

    // キューに追加（優先度に応じたキュー）
    const job = await this.queues[request.priority].add(
      request.type,
      { ...request, channels },
      {
        priority: this.getPriorityValue(request.priority),
        attempts: 3,
        backoff: { type: 'exponential', delay: 1_000 },
        removeOnComplete: { age: 86400 }, // 24時間後に完了ジョブを削除
      }
    );

    // 重複防止キーを設定（24時間）
    await redis.set(dedupKey, job.id!, { EX: 86400 });

    return job.id!;
  }

  private async resolveChannels(
    request: NotificationRequest,
    prefs: UserNotificationPreferences
  ): Promise<NotificationChannel[]> {
    if (request.channels) return request.channels.filter(ch => prefs.channels[ch] !== false);

    // 優先度からデフォルトチャネルを決定
    const defaults: Record<NotificationPriority, NotificationChannel[]> = {
      critical: ['sms', 'slack', 'email'],
      high:     ['email', 'push'],
      medium:   ['push'],
      low:      ['email'],
    };

    return defaults[request.priority].filter(ch => {
      // ユーザー設定チェック
      if (prefs.channels[ch] === false) return false;
      if (prefs.typeSettings[request.type]?.[ch] === false) return false;
      // 低優先度はマーケティング同意が必要
      if (request.priority === 'low' && !prefs.marketingConsent) return false;
      return true;
    });
  }

  private getPriorityValue(priority: NotificationPriority): number {
    return { critical: 1, high: 2, medium: 3, low: 4 }[priority];
  }
}
```

```typescript
// src/notifications/channels/emailChannel.ts

export class EmailChannel {
  private ses: SESv2Client;

  async send(params: {
    to: string;
    subject: string;
    templateId: string;
    data: Record<string, any>;
  }): Promise<void> {
    // テンプレートHTMLを生成
    const html = await renderEmailTemplate(params.templateId, params.data);
    const text = htmlToText(html);

    await this.ses.send(new SendEmailCommand({
      Destination: { ToAddresses: [params.to] },
      Content: {
        Simple: {
          Subject: { Data: params.subject, Charset: 'UTF-8' },
          Body: {
            Html: { Data: html, Charset: 'UTF-8' },
            Text: { Data: text, Charset: 'UTF-8' },
          },
        },
      },
      FromEmailAddress: `${process.env.EMAIL_FROM_NAME} <${process.env.EMAIL_FROM_ADDRESS}>`,
      // 配信停止ヘッダー（CAN-SPAM対応）
      ListManagementOptions: {
        ContactListName: 'myapp-users',
        TopicName: params.templateId,
      },
    }));
  }
}
```

```typescript
// src/notifications/channels/pushChannel.ts — FCMプッシュ通知

export class PushChannel {
  async send(params: {
    userId: string;
    title: string;
    body: string;
    data?: Record<string, string>;
    imageUrl?: string;
  }): Promise<void> {
    // ユーザーのデバイストークンを取得（複数デバイス対応）
    const devices = await prisma.deviceToken.findMany({
      where: { userId: params.userId, active: true },
    });

    if (devices.length === 0) return;

    // 静寂時間チェック（22時〜8時はプッシュ送信しない）
    const hour = new Date().getHours();
    if (hour >= 22 || hour < 8) {
      // スケジュール通知として8時に遅延送信
      await scheduleNotification(params, new Date().setHours(8, 0, 0, 0));
      return;
    }

    // FCMマルチキャスト送信
    const message = {
      notification: { title: params.title, body: params.body, imageUrl: params.imageUrl },
      data: params.data ?? {},
      tokens: devices.map(d => d.token),
      android: { priority: 'high' as const },
      apns: { payload: { aps: { sound: 'default' } } },
    };

    const response = await firebaseAdmin.messaging().sendEachForMulticast(message);

    // 無効なトークンを無効化
    for (const [i, result] of response.responses.entries()) {
      if (!result.success && result.error?.code === 'messaging/registration-token-not-registered') {
        await prisma.deviceToken.update({
          where: { id: devices[i].id },
          data: { active: false },
        });
      }
    }
  }
}
```

```typescript
// src/notifications/worker.ts — 通知ワーカー

// 各優先度キューのワーカー
['critical', 'high', 'medium', 'low'].forEach(priority => {
  const worker = new Worker(
    `notifications:${priority}`,
    async (job: Job) => {
      const { userId, type, channels, data } = job.data;
      const user = await prisma.user.findUniqueOrThrow({ where: { id: userId } });

      const results = await Promise.allSettled(
        channels.map(async (channel: NotificationChannel) => {
          switch (channel) {
            case 'email':
              await emailChannel.send({
                to: user.email,
                subject: getEmailSubject(type, data),
                templateId: type,
                data: { user, ...data },
              });
              break;
            case 'push':
              await pushChannel.send({ userId, title: getPushTitle(type), body: getPushBody(type, data) });
              break;
            case 'slack':
              await slackChannel.send({ userId, type, data });
              break;
          }
        })
      );

      // 配信ログを保存
      await prisma.notificationLog.create({
        data: { userId, type, channels, results: JSON.stringify(results) },
      });
    },
    { connection: redis, concurrency: priority === 'critical' ? 10 : 3 }
  );
});
```

---

## まとめ

Claude Codeでマルチチャネル通知システムを設計する：

1. **CLAUDE.md** に優先度ごとのチャネル・重複防止24時間・静寂時間・レート制限を明記
2. **優先度別キュー** でcriticalは即時、lowはバッチ処理（リソース効率化）
3. **deduplicationKey** でRedis TTL24時間内の重複送信を防止
4. **静寂時間チェック** でプッシュ通知を8時にスケジュール遅延

---

*通知システム設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
