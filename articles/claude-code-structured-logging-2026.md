---
title: "Claude Codeで構造化ログを設計する：pino・リクエストID・ログレベル戦略"
emoji: "📋"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "pino", "observability"]
published: true
---

## はじめに

`console.log('error happened')` では本番障害で調査ができない——リクエストIDがない、ログレベルがない、JSON形式でない。Claude Codeにpinoを使った構造化ログ設計を生成させる。

---

## CLAUDE.mdにロギングルールを書く

```markdown
## ロギング設計ルール

### ロガー
- pino使用（高速・JSON出力・本番向け）
- console.log/console.error禁止（logger.info/errorを使う）
- pino-httpでHTTPリクエストを自動ロギング

### リクエストID
- 全リクエストにX-Request-IDを付与（クライアントから送られた場合は引き継ぎ）
- 全ログにrequest_idを含める（トレーサビリティ）
- レスポンスヘッダーにもX-Request-IDを返す

### ログレベル
- trace: デバッグ詳細（開発のみ）
- debug: 開発用詳細
- info: 正常系の重要イベント（ユーザー作成、注文完了等）
- warn: 回復可能な異常（認証失敗、バリデーションエラー）
- error: アプリエラー（予期しない例外、外部サービス障害）
- fatal: システム停止が必要なエラー

### 機密情報
- パスワード・トークン・クレカ情報をログに出力禁止
- pinoのredactオプションで自動除外
```

---

## 構造化ログの生成

```
pinoを使った構造化ログ設計を生成してください。

要件：
- JSON形式でログ出力
- X-Request-IDをリクエスト全体で引き継ぎ
- pino-httpでHTTPログ自動出力
- 機密フィールドを自動マスク
- 本番はlevel=info、開発はlevel=debug

生成ファイル:
- src/logger.ts
- src/middleware/requestId.ts
```

---

## 生成される構造化ログ

```typescript
// src/logger.ts
import pino from 'pino';

const isDev = process.env.NODE_ENV !== 'production';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? (isDev ? 'debug' : 'info'),

  // 機密フィールドをマスク
  redact: {
    paths: [
      'password',
      'passwordHash',
      'token',
      'accessToken',
      'refreshToken',
      'authorization',
      'req.headers.authorization',
      'creditCard',
      '*.creditCard',
    ],
    censor: '[REDACTED]',
  },

  // 開発環境: 人間が読みやすい形式
  ...(isDev && {
    transport: {
      target: 'pino-pretty',
      options: {
        colorize: true,
        translateTime: 'HH:MM:ss',
        ignore: 'pid,hostname',
      },
    },
  }),
});
```

```typescript
// src/middleware/requestId.ts
import { randomUUID } from 'crypto';

export const requestIdMiddleware = (req: Request, res: Response, next: NextFunction) => {
  // クライアントから送られたIDを引き継ぐ（なければ生成）
  const requestId = (req.headers['x-request-id'] as string) ?? randomUUID();

  req.id = requestId;
  res.setHeader('X-Request-ID', requestId); // レスポンスにも付与

  next();
};
```

```typescript
// src/app.ts
import pinoHttp from 'pino-http';
import { requestIdMiddleware } from './middleware/requestId';

const app = express();

// リクエストIDを最初に付与
app.use(requestIdMiddleware);

// pino-httpでHTTPリクエストを自動ロギング
app.use(pinoHttp({
  logger,
  genReqId: (req) => req.id, // 既に付与したIDを使う

  // ヘルスチェックエンドポイントのログを抑制
  autoLogging: {
    ignore: (req) => req.url === '/health',
  },

  // ログに含めるカスタムフィールド
  customAttributeKeys: { req: 'request', res: 'response' },
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      requestId: req.id,
    }),
  },
}));
```

---

## コンテキスト付きロガー

```typescript
// リクエストコンテキストを持つロガーを作成
function getRequestLogger(req: Request) {
  return logger.child({
    requestId: req.id,
    userId: (req as AuthenticatedRequest).user?.id,
    tenantId: (req as AuthenticatedRequest).user?.tenantId,
  });
}

// 使用例
router.post('/orders', authenticate, async (req, res) => {
  const log = getRequestLogger(req);

  log.info({ body: req.body }, 'Creating order');

  const order = await createOrder(req.body);

  log.info({ orderId: order.id }, 'Order created successfully');

  res.status(201).json(order);
});
```

---

## ログ出力例（JSON）

```json
{
  "level": 30,
  "time": 1710123456789,
  "msg": "Order created successfully",
  "requestId": "a3c2e1b4-8f7d-4e9a-b2c1-5d4e3f2a1b0c",
  "userId": "user_123",
  "tenantId": "tenant_abc",
  "orderId": "order_xyz",
  "hostname": "app-pod-1"
}
```

---

## まとめ

Claude Codeで構造化ログを設計する：

1. **CLAUDE.md** にpino必須・console禁止・リクエストID必須・機密フィールド除外を明記
2. **X-Request-ID** でリクエスト全体のログを追跡可能にする
3. **pino-http** でHTTPログを自動出力
4. **redact** で機密フィールドを自動マスク

---

*ロギングのレビュー（機密情報漏洩、リクエストID欠落）は **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
