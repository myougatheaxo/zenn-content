---
title: "Claude Codeでメール送信を設計する：SES・テンプレート・キュー処理"
emoji: "📧"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "bullmq"]
published: true
---

## はじめに

メール送信をAPIレスポンス内で同期処理すると、SESの遅延でリクエストがタイムアウトする。バウンスを無視すると送信レートが下がる。Claude Codeに安全なメール送信設計を生成させる。

---

## CLAUDE.mdにメール送信ルールを書く

```markdown
## メール送信設計ルール

### 非同期処理（必須）
- メール送信はBullMQキューに投げる（APIレスポンスをブロックしない）
- リトライ: 最大3回、指数バックオフ（1min→5min→30min）
- 送信履歴をDBに記録（べき等性確保）

### テンプレート
- HTML + テキスト両方を用意（メールクライアント互換性）
- テンプレートエンジン: Handlebars（型安全なパラメータ）
- インライン CSS（多くのメールクライアントはstyleタグ非対応）

### セキュリティ
- 送信元: noreply@example.com（送信専用、返信不可）
- ユーザー入力をHTMLエスケープ（XSS対策）
- バウンス/苦情をSNSで受け取りDB更新

### プロバイダ
- 本番: AWS SES（安価・高信頼）
- 開発: Mailhog or Mailtrap（実際に送らない）
```

---

## メール送信の生成

```
メール送信システムを設計してください。

ユースケース:
- ユーザー登録完了メール
- パスワードリセットメール
- 注文確認メール

要件：
- BullMQで非同期送信
- AWS SES使用
- Handlebarsテンプレート
- 送信履歴DB記録
- バウンス処理

生成ファイル:
- src/email/emailService.ts
- src/email/templates/
- src/workers/emailWorker.ts
```

---

## 生成されるメール送信サービス

```typescript
// src/email/emailService.ts
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses';
import Handlebars from 'handlebars';
import { readFileSync } from 'fs';
import { join } from 'path';

const ses = new SESClient({ region: process.env.AWS_REGION });

const FROM_ADDRESS = process.env.EMAIL_FROM ?? 'noreply@example.com';

// テンプレートをキャッシュ
const templateCache = new Map<string, HandlebarsTemplateDelegate>();

function getTemplate(name: string): HandlebarsTemplateDelegate {
  if (!templateCache.has(name)) {
    const html = readFileSync(join(__dirname, 'templates', `${name}.html`), 'utf-8');
    templateCache.set(name, Handlebars.compile(html));
  }
  return templateCache.get(name)!;
}

export async function sendEmailDirect(params: {
  to: string;
  subject: string;
  template: string;
  data: Record<string, unknown>;
  idempotencyKey?: string;
}): Promise<void> {
  // 重複送信チェック
  if (params.idempotencyKey) {
    const existing = await prisma.emailLog.findUnique({
      where: { idempotencyKey: params.idempotencyKey },
    });
    if (existing) {
      logger.info({ idempotencyKey: params.idempotencyKey }, 'Email already sent, skipping');
      return;
    }
  }

  const template = getTemplate(params.template);
  const html = template(params.data);

  const command = new SendEmailCommand({
    Source: FROM_ADDRESS,
    Destination: { ToAddresses: [params.to] },
    Message: {
      Subject: { Data: params.subject, Charset: 'UTF-8' },
      Body: {
        Html: { Data: html, Charset: 'UTF-8' },
      },
    },
  });

  await ses.send(command);

  // 送信履歴を記録
  if (params.idempotencyKey) {
    await prisma.emailLog.create({
      data: {
        idempotencyKey: params.idempotencyKey,
        to: params.to,
        subject: params.subject,
        template: params.template,
        sentAt: new Date(),
      },
    });
  }
}
```

```typescript
// src/email/emailQueue.ts
// API側: キューに投げるだけ（非同期）
export async function queueWelcomeEmail(userId: string, email: string, name: string): Promise<void> {
  await emailQueue.add(
    'welcome',
    { userId, email, name, template: 'welcome' },
    {
      attempts: 3,
      backoff: { type: 'exponential', delay: 60000 }, // 1min→5min→30min
      jobId: `welcome-${userId}`, // 重複防止
    }
  );
}

export async function queuePasswordResetEmail(
  email: string,
  resetToken: string
): Promise<void> {
  await emailQueue.add(
    'password-reset',
    { email, resetToken, template: 'password-reset' },
    {
      attempts: 3,
      backoff: { type: 'exponential', delay: 60000 },
      jobId: `reset-${email}-${Date.now()}`,
    }
  );
}
```

```typescript
// src/workers/emailWorker.ts
const emailWorker = new Worker('email', async (job) => {
  const { email, template, name, resetToken } = job.data;

  const templateData: Record<string, unknown> = {
    name,
    year: new Date().getFullYear(),
    appUrl: process.env.APP_URL,
    ...(resetToken && { resetUrl: `${process.env.APP_URL}/reset-password?token=${resetToken}` }),
  };

  await sendEmailDirect({
    to: email,
    subject: getSubjectForTemplate(template),
    template,
    data: templateData,
    idempotencyKey: job.id,
  });
}, {
  connection: redisConnection,
  concurrency: 5, // 並列5件まで
});

emailWorker.on('failed', (job, err) => {
  logger.error({ jobId: job?.id, err }, 'Email job failed');
});
```

---

## まとめ

Claude Codeでメール送信を設計する：

1. **CLAUDE.md** に非同期必須・リトライ設定・テンプレート方式を明記
2. **BullMQキュー** でAPIレスポンスをブロックしない
3. **べき等性キー** で重複送信を防ぐ
4. **Handlebarsテンプレート** でHTML/テキスト両方を型安全に生成

---

*メール送信の設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
