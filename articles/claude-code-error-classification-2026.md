---
title: "Claude Codeでエラー分類システムを設計する：運用エラーvsプログラマーエラー・エラーカタログ・一貫したAPIエラーレスポンス"
emoji: "🚨"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-19 10:00"
---

## はじめに

「エラーハンドリングがバラバラでAPIが一貫したエラーを返さない」「500エラーと400エラーが混在していて原因が追えない」——エラーを体系的に分類し、一貫したエラーレスポンスとロギングを設計をClaude Codeに生成させる。

---

## CLAUDE.mdにエラー分類設計ルールを書く

```markdown
## エラー分類設計ルール

### エラーの種類
- Operational Error（運用エラー）: 予期される失敗（バリデーション、認証失敗、NotFound）
  → ユーザーにエラーを返す。ログはWARNレベル
- Programmer Error（プログラマーエラー）: バグ（NullPointer、型エラー）
  → 500エラー。ログはERRORレベル。Sentryに通知

### エラーカタログ
- エラーコードを定義（例: USER_NOT_FOUND, INSUFFICIENT_BALANCE）
- コード、HTTPステータス、メッセージのマッピング
- エラーコードはドキュメント化してフロントエンドと共有

### レスポンス形式
- { code: string, message: string, details?: object }
- エラーコードはアルファベット大文字スネークケース
- 内部エラーの詳細は本番で非表示（devのみ表示）
```

---

## エラー分類システム実装の生成

```
エラー分類システムを設計してください。

要件：
- Operational/Programmerエラーの分離
- エラーカタログ
- 一貫したAPIエラーレスポンス
- エラーミドルウェア

生成ファイル: src/errors/
```

---

## 生成されるエラー分類システム実装

```typescript
// src/errors/appError.ts — アプリケーションエラー基底

export class AppError extends Error {
  readonly isOperational: boolean;  // true = 運用エラー、false = プログラマーエラー

  constructor(
    message: string,
    readonly code: string,
    readonly statusCode: number,
    readonly details?: Record<string, unknown>,
    isOperational = true
  ) {
    super(message);
    this.name = this.constructor.name;
    this.isOperational = isOperational;
    Error.captureStackTrace(this, this.constructor);
  }
}

// エラーカタログ定義
export const ErrorCatalog = {
  // 認証・認可 (401, 403)
  UNAUTHORIZED:           { code: 'UNAUTHORIZED', status: 401, message: '認証が必要です' },
  FORBIDDEN:              { code: 'FORBIDDEN', status: 403, message: 'この操作を行う権限がありません' },
  INVALID_TOKEN:          { code: 'INVALID_TOKEN', status: 401, message: 'トークンが無効または期限切れです' },

  // リソース (404, 409)
  NOT_FOUND:              { code: 'NOT_FOUND', status: 404, message: 'リソースが見つかりません' },
  ALREADY_EXISTS:         { code: 'ALREADY_EXISTS', status: 409, message: 'リソースが既に存在します' },
  CONFLICT:               { code: 'CONFLICT', status: 409, message: 'データが競合しています' },

  // バリデーション (400)
  VALIDATION_ERROR:       { code: 'VALIDATION_ERROR', status: 400, message: '入力値が正しくありません' },
  INVALID_PARAMETER:      { code: 'INVALID_PARAMETER', status: 400, message: 'パラメーターが無効です' },

  // ビジネスロジック (422)
  INSUFFICIENT_BALANCE:   { code: 'INSUFFICIENT_BALANCE', status: 422, message: '残高が不足しています' },
  OUT_OF_STOCK:           { code: 'OUT_OF_STOCK', status: 422, message: '在庫が不足しています' },
  INVALID_STATE:          { code: 'INVALID_STATE', status: 422, message: '現在の状態では操作できません' },
  PAYMENT_FAILED:         { code: 'PAYMENT_FAILED', status: 422, message: '決済に失敗しました' },

  // レート制限 (429)
  RATE_LIMIT_EXCEEDED:    { code: 'RATE_LIMIT_EXCEEDED', status: 429, message: 'リクエスト制限を超えました' },

  // サービス利用不可 (503)
  SERVICE_UNAVAILABLE:    { code: 'SERVICE_UNAVAILABLE', status: 503, message: 'サービスが一時的に利用できません' },
} as const;

// エラーファクトリー
export function createError(
  catalog: (typeof ErrorCatalog)[keyof typeof ErrorCatalog],
  details?: Record<string, unknown>
): AppError {
  return new AppError(catalog.message, catalog.code, catalog.status, details);
}

// 具体的なエラークラス
export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    super(
      `${resource}${id ? ` (${id})` : ''} not found`,
      ErrorCatalog.NOT_FOUND.code,
      ErrorCatalog.NOT_FOUND.status,
      { resource, id }
    );
  }
}

export class ValidationError extends AppError {
  constructor(violations: Array<{ field: string; message: string }>) {
    super(
      ErrorCatalog.VALIDATION_ERROR.message,
      ErrorCatalog.VALIDATION_ERROR.code,
      ErrorCatalog.VALIDATION_ERROR.status,
      { violations }
    );
  }
}

export class ConflictError extends AppError {
  constructor(message: string, details?: Record<string, unknown>) {
    super(message, ErrorCatalog.CONFLICT.code, ErrorCatalog.CONFLICT.status, details);
  }
}

export class InsufficientBalanceError extends AppError {
  constructor(required: Money, available: Money) {
    super(
      ErrorCatalog.INSUFFICIENT_BALANCE.message,
      ErrorCatalog.INSUFFICIENT_BALANCE.code,
      ErrorCatalog.INSUFFICIENT_BALANCE.status,
      { required: required.toValue(), available: available.toValue() }
    );
  }
}
```

```typescript
// src/errors/errorMiddleware.ts — Expressエラーミドルウェア

export interface ErrorResponse {
  code: string;
  message: string;
  details?: unknown;
  requestId?: string;
  // 本番では非表示
  stack?: string;
}

export function errorMiddleware(
  error: Error,
  req: Request,
  res: Response,
  _next: NextFunction
): void {
  const requestId = req.headers['x-request-id'] as string ?? ulid();

  if (error instanceof AppError && error.isOperational) {
    // 運用エラー: ユーザーに詳細を返す
    logger.warn({
      requestId,
      code: error.code,
      statusCode: error.statusCode,
      url: req.url,
      method: req.method,
    }, `Operational error: ${error.message}`);

    const response: ErrorResponse = {
      code: error.code,
      message: error.message,
      requestId,
    };

    if (error.details) {
      response.details = error.details;
    }

    res.status(error.statusCode).json(response);
    return;
  }

  // プログラマーエラー / 未知のエラー: 詳細を隠して500を返す
  logger.error({
    requestId,
    error: error.message,
    stack: error.stack,
    url: req.url,
    method: req.method,
  }, 'Unexpected error');

  // エラー監視サービスに通知
  sentryClient.captureException(error, { extra: { requestId } });

  const response: ErrorResponse = {
    code: 'INTERNAL_SERVER_ERROR',
    message: 'An unexpected error occurred',
    requestId,
    // 開発環境のみスタックトレースを含める
    ...(process.env.NODE_ENV === 'development' ? { stack: error.stack } : {}),
  };

  res.status(500).json(response);
}

// Zodバリデーションエラーの変換
export function zodErrorToValidationError(zodError: ZodError): ValidationError {
  const violations = zodError.issues.map(issue => ({
    field: issue.path.join('.'),
    message: issue.message,
  }));
  return new ValidationError(violations);
}

// 使用例
app.post('/api/orders', async (req, res, next) => {
  try {
    const input = OrderSchema.parse(req.body); // Zodバリデーション
    const order = await orderService.create(input);
    res.status(201).json(order);
  } catch (error) {
    if (error instanceof ZodError) {
      next(zodErrorToValidationError(error));  // ValidationErrorに変換
    } else {
      next(error);  // その他はミドルウェアに委ねる
    }
  }
});
```

---

## まとめ

Claude Codeでエラー分類システムを設計する：

1. **CLAUDE.md** に運用エラー（予期される失敗）はWARNログ＋詳細をユーザーに返す・プログラマーエラーはERRORログ＋Sentry通知＋500で詳細を隠すを明記
2. **エラーカタログ** でcode/status/messageをセットで定義——フロントエンドが`response.code === 'INSUFFICIENT_BALANCE'`で判定してUIに「残高不足」と表示できる
3. **`isOperational`フラグ** で運用エラーとバグを判別——`if (error instanceof AppError && error.isOperational)`で安全にユーザーへのレスポンスを決定。不明なエラーは全て500
4. **requestIdでエラーを追跡** ——ユーザーが「requestId: `01H...`でエラーが出た」と言えば、ログから原因を追跡。`X-Request-Id`ヘッダーで連携

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
