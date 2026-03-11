---
title: "Claude CodeでコマンドバスパターンをCQRSに適用する：コマンドハンドラー・ミドルウェアチェーン・バリデーション"
emoji: "📨"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-20 16:00"
---

## はじめに

「ユースケースが増えるたびにコントローラーが肥大化する」「バリデーション・ロギング・認可が各ユースケースに散在している」——コマンドバスでコマンド実行を統一化し、横断的関心事をミドルウェアに集約する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにコマンドバスパターン設計ルールを書く

```markdown
## コマンドバスパターン設計ルール

### コマンドの設計
- 1コマンド = 1ユースケース（意図を名前で表現）
- コマンドはデータのみ（メソッドなし）、バリデーションは別層
- 命名: PlaceOrderCommand, CancelOrderCommand（過去形でなく命令形）

### コマンドバス
- コマンドをハンドラーにルーティング
- ミドルウェアチェーンで横断的関心事を処理
  - バリデーション → 認可 → ロギング → トランザクション → ハンドラー

### コマンドハンドラー
- 1コマンドにつき1ハンドラー
- インフラに依存しない（ポートのみに依存）
- handle()メソッドのみを公開
```

---

## コマンドバス実装の生成

```
コマンドバスパターンを設計してください。

要件：
- 型安全なコマンド登録
- ミドルウェアチェーン
- Zodバリデーション統合
- トランザクションミドルウェア

生成ファイル: src/application/commandBus/
```

---

## 生成されるコマンドバス実装

```typescript
// src/application/commandBus/commandBus.ts — コマンドバスコア

// コマンドの基底型（タグ付き型で型安全なルーティング）
export interface Command<TResult = void> {
  readonly _commandType: string;
  readonly _resultType?: TResult;
}

// コマンドハンドラーインターフェース
export interface ICommandHandler<TCommand extends Command<TResult>, TResult = void> {
  handle(command: TCommand): Promise<TResult>;
}

// ミドルウェア型
export type CommandMiddleware = <TResult>(
  command: Command<TResult>,
  next: () => Promise<TResult>
) => Promise<TResult>;

// コマンドバス実装
export class CommandBus {
  private readonly handlers = new Map<string, ICommandHandler<any, any>>();
  private readonly middlewares: CommandMiddleware[] = [];

  // ハンドラーを登録
  register<TCommand extends Command<TResult>, TResult>(
    commandType: string,
    handler: ICommandHandler<TCommand, TResult>
  ): void {
    this.handlers.set(commandType, handler);
  }

  // ミドルウェアを追加（先に追加したものが外側）
  use(middleware: CommandMiddleware): void {
    this.middlewares.push(middleware);
  }

  // コマンドを実行
  async dispatch<TResult>(command: Command<TResult>): Promise<TResult> {
    const handler = this.handlers.get(command._commandType);
    if (!handler) {
      throw new Error(`No handler registered for command: ${command._commandType}`);
    }

    // ミドルウェアチェーンを構築（最後に追加したミドルウェアが内側）
    const chain = this.middlewares.reduceRight<() => Promise<TResult>>(
      (next, middleware) => () => middleware(command, next),
      () => handler.handle(command)
    );

    return chain();
  }
}
```

```typescript
// src/application/commandBus/middlewares/ — ミドルウェア群

// 1. バリデーションミドルウェア（Zod統合）
const validationSchemas = new Map<string, ZodSchema>();

export function registerSchema<T>(commandType: string, schema: ZodSchema<T>): void {
  validationSchemas.set(commandType, schema);
}

export const validationMiddleware: CommandMiddleware = async (command, next) => {
  const schema = validationSchemas.get(command._commandType);
  if (schema) {
    const result = schema.safeParse(command);
    if (!result.success) {
      throw new ValidationError(
        result.error.issues.map(i => ({ field: i.path.join('.'), message: i.message }))
      );
    }
  }
  return next();
};

// 2. ロギングミドルウェア
export const loggingMiddleware: CommandMiddleware = async (command, next) => {
  const start = Date.now();
  logger.info({ commandType: command._commandType }, 'Command dispatched');
  try {
    const result = await next();
    logger.info({ commandType: command._commandType, durationMs: Date.now() - start }, 'Command succeeded');
    return result;
  } catch (error) {
    logger.error({ commandType: command._commandType, error, durationMs: Date.now() - start }, 'Command failed');
    throw error;
  }
};

// 3. トランザクションミドルウェア（Prismaトランザクション自動管理）
export const transactionMiddleware: CommandMiddleware = async (command, next) => {
  // コマンドにトランザクションが不要な場合はスキップ
  if ((command as any)._noTransaction) return next();

  return prisma.$transaction(async (tx) => {
    // AsyncLocalStorageでトランザクションを伝播
    return transactionContext.run(tx, () => next());
  });
};

// 4. 認可ミドルウェア
type AuthorizationRule = (command: Command, user: AuthUser) => boolean;
const authRules = new Map<string, AuthorizationRule>();

export function registerAuthRule(commandType: string, rule: AuthorizationRule): void {
  authRules.set(commandType, rule);
}

export const authorizationMiddleware = (getUser: () => AuthUser | undefined): CommandMiddleware =>
  async (command, next) => {
    const rule = authRules.get(command._commandType);
    if (rule) {
      const user = getUser();
      if (!user) throw new UnauthorizedError();
      if (!rule(command, user)) throw new ForbiddenError();
    }
    return next();
  };
```

```typescript
// src/application/commands/placeOrderCommand.ts — コマンド定義

export interface PlaceOrderCommand extends Command<PlaceOrderResult> {
  readonly _commandType: 'PlaceOrderCommand';
  readonly userId: string;
  readonly items: Array<{ productId: string; quantity: number; unitPrice: number }>;
  readonly webhookUrl?: string;
}

export interface PlaceOrderResult {
  orderId: string;
  total: number;
}

// コマンドファクトリー関数
export const createPlaceOrderCommand = (input: Omit<PlaceOrderCommand, '_commandType' | '_resultType'>): PlaceOrderCommand => ({
  _commandType: 'PlaceOrderCommand',
  ...input,
});

// Zodスキーマ（バリデーションミドルウェアで使用）
export const PlaceOrderSchema = z.object({
  userId: z.string().ulid(),
  items: z.array(z.object({
    productId: z.string().min(1),
    quantity: z.number().int().positive().max(100),
    unitPrice: z.number().positive(),
  })).min(1).max(50),
  webhookUrl: z.string().url().optional(),
});

registerSchema('PlaceOrderCommand', PlaceOrderSchema);
registerAuthRule('PlaceOrderCommand', (cmd, user) => user.id === (cmd as PlaceOrderCommand).userId);
```

```typescript
// src/application/handlers/placeOrderHandler.ts — コマンドハンドラー

export class PlaceOrderHandler implements ICommandHandler<PlaceOrderCommand, PlaceOrderResult> {
  constructor(
    private readonly orderRepository: IOrderRepository,
    private readonly paymentGateway: IPaymentGateway,
    private readonly eventPublisher: IEventPublisher
  ) {}

  async handle(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    const order = Order.create(
      command.userId,
      command.items.map(i => ({
        productId: i.productId,
        quantity: i.quantity,
        price: Money.of(i.unitPrice, 'JPY'),
      }))
    );

    await this.paymentGateway.createPaymentIntent(order.total, command.userId);
    order.submit();
    await this.orderRepository.save(order);
    await this.eventPublisher.publish({ eventType: 'OrderPlaced', orderId: order.id });

    return { orderId: order.id, total: order.total.amount };
  }
}

// ===== コンポジションルート（main.ts） =====

const commandBus = new CommandBus();

// ミドルウェア登録（順番が重要: 外→内）
commandBus.use(loggingMiddleware);
commandBus.use(validationMiddleware);
commandBus.use(authorizationMiddleware(() => currentUserContext.get()));
commandBus.use(transactionMiddleware);

// ハンドラー登録
commandBus.register('PlaceOrderCommand', new PlaceOrderHandler(orderRepo, stripeGateway, eventBus));
commandBus.register('CancelOrderCommand', new CancelOrderHandler(orderRepo, eventBus));
commandBus.register('UpdateShippingCommand', new UpdateShippingHandler(orderRepo));

// HTTPコントローラー（薄い層）
app.post('/api/orders', requireAuth, async (req, res, next) => {
  try {
    const result = await commandBus.dispatch(
      createPlaceOrderCommand({ userId: req.user.id, ...req.body })
    );
    res.status(201).json(result);
  } catch (error) {
    next(error);
  }
});
```

---

## まとめ

Claude Codeでコマンドバスパターンを設計する：

1. **CLAUDE.md** に1コマンド=1ユースケース・ミドルウェアで横断的関心事を集約（バリデーション→認可→ログ→トランザクション）・コントローラーはdispatch()を呼ぶだけを明記
2. **型安全なルーティング** ——`_commandType`文字列でハンドラーを特定。`dispatch(command)`の戻り値型は`Command<TResult>`の`TResult`に自動推論される
3. **ミドルウェアチェーンで横断的関心事を分離** ——バリデーション・認可・ロギング・トランザクションを各ハンドラーに書かない。新しいコマンドを追加してもこれらは自動的に適用される
4. **`registerSchema()`でコマンドにバリデーションを宣言的に付与** ——`PlaceOrderSchema`をコマンド定義ファイルに書くだけで、バスを通るたびに自動バリデーション。ハンドラーにZodコードは一行もない

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
