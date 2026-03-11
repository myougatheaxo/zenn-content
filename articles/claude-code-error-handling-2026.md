---
title: "Claude Codeでエラーハンドリングを設計する：カスタムエラークラスと一元管理"
emoji: "🚨"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "express", "pino"]
published: true
---

## はじめに

エラーハンドリングを各ルートで個別実装すると、レスポンス形式がバラバラになる。認証エラーが500になる。スタックトレースがクライアントに漏れる。Claude Codeに一元管理された設計を生成させる。

---

## CLAUDE.mdにエラーハンドリングルールを書く

```markdown
## エラーハンドリングルール

### エラークラス
- AppError基底クラス（statusCode, code, isOperational）
- 用途別サブクラス: NotFoundError(404), UnauthorizedError(401), ForbiddenError(403),
  ValidationError(422), ConflictError(409), RateLimitError(429)
- isOperational=falseのエラーは予期しないバグ（Slackで緊急通知）

### クライアントへのレスポンス
- 本番: スタックトレースを返さない
- 開発: stack情報を含める
- 全エラーに error.code を含める（クライアントが機械的に処理できる）
- 形式: { error: { code: string, message: string, details?: any } }

### ロギング
- 4xx: warn レベル（オペレーショナルエラー）
- 5xx: error レベル + Slack通知
- 全エラーにrequest IDを含める
```

---

## エラーハンドリングの生成

```
カスタムエラークラスと一元管理されたエラーハンドラーを設計してください。

要件：
- AppError基底クラスと用途別サブクラス
- Expressのエラーハンドラーミドルウェア
- 本番でスタックトレースを隠す
- isOperational=falseはSlack緊急通知
- pino-httpでリクエストIDとエラーをログ

生成ファイル:
- src/errors/AppError.ts
- src/middleware/errorHandler.ts
```

---

## 生成されるエラーハンドリング

```typescript
// src/errors/AppError.ts
export class AppError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly code: string,
    message: string,
    public readonly isOperational: boolean = true,
    public readonly details?: unknown
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    super(404, 'NOT_FOUND', `${resource}${id ? ` (${id})` : ''} not found`);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') {
    super(401, 'UNAUTHORIZED', message);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Insufficient permissions') {
    super(403, 'FORBIDDEN', message);
  }
}

export class ValidationError extends AppError {
  constructor(details: Array<{ field: string; message: string }>) {
    super(422, 'VALIDATION_ERROR', 'Validation failed', true, details);
  }
}

export class ConflictError extends AppError {
  constructor(resource: string, message?: string) {
    super(409, 'CONFLICT', message ?? `${resource} already exists`);
  }
}

export class RateLimitError extends AppError {
  constructor(retryAfter: number) {
    super(429, 'RATE_LIMIT_EXCEEDED', `Rate limit exceeded. Retry after ${retryAfter}s`, true, { retryAfter });
  }
}
```

```typescript
// src/middleware/errorHandler.ts
import { AppError } from '../errors/AppError';

const isDev = process.env.NODE_ENV === 'development';

export const errorHandler = async (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> => {
  if (err instanceof AppError) {
    // オペレーショナルエラー（予測された異常系）
    if (err.statusCode >= 500) {
      logger.error({ err, requestId: req.id }, err.message);
      await notifySlack(`🚨 Server Error: ${err.message}\nCode: ${err.code}\nURL: ${req.method} ${req.url}`);
    } else {
      logger.warn({ err, requestId: req.id }, err.message);
    }

    res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        ...(err.details !== undefined && { details: err.details }),
        ...(isDev && { stack: err.stack }),
      },
    });
    return;
  }

  // 予期しないエラー（バグ）
  logger.error({ err, requestId: req.id }, 'Unexpected error');
  await notifySlack(`🔥 Unexpected Error: ${err.message}\nURL: ${req.method} ${req.url}\nStack: ${err.stack?.substring(0, 500)}`);

  res.status(500).json({
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: isDev ? err.message : 'An unexpected error occurred',
      ...(isDev && { stack: err.stack }),
    },
  });
};

// 未処理のPromise拒否をキャッチ
process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'Unhandled promise rejection');
  // グレースフルシャットダウン
  process.exit(1);
});
```

```typescript
// 使用例
router.get('/users/:id', authenticate, async (req, res) => {
  const user = await prisma.user.findUnique({ where: { id: req.params.id } });

  if (!user) {
    throw new NotFoundError('User', req.params.id); // Expressのnext(err)不要
  }

  res.json(user);
});
```

---

## 非同期エラーをExpressに伝播

```typescript
// Express 4ではasync/awaitのエラーはnext()が必要
// asyncMiddlewareラッパーで自動化
export function asyncHandler(fn: AsyncRequestHandler) {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// または express-async-errors パッケージで自動適用
import 'express-async-errors'; // これだけでexpress 4でもthrowが動く
```

---

## まとめ

Claude Codeでエラーハンドリングを設計する：

1. **CLAUDE.md** にカスタムエラークラス・レスポンス形式・ロギングレベルを明記
2. **AppError階層** で用途別エラーを型安全に使う
3. **errorHandler** でスタックトレース隠蔽・Slack通知を一元管理
4. **isOperational** でバグと予測エラーを区別して通知レベルを変える

---

*エラーハンドリングのレビュー（スタックトレース漏洩、一元化されていないエラー処理）は **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
