---
title: "Claude CodeでDIコンテナを設計する：依存性注入・テスト可能な設計・スコープ管理"
emoji: "💉"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "testing", "architecture"]
published: true
published_at: "2026-03-16 15:00"
---

## はじめに

「テストのためにモックを渡せない」「グローバルなサービスインスタンスが増えて依存関係が追えない」——依存性注入（DI）コンテナでモジュール間の依存を明示化し、テスト可能な設計をClaude Codeに生成させる。

---

## CLAUDE.mdにDI設計ルールを書く

```markdown
## 依存性注入（DI）設計ルール

### 原則
- new ServiceName() を直接呼び出さない（DIコンテナから取得）
- サービスの依存は全てコンストラクタで宣言（コンストラクタインジェクション）
- テスト時はモック実装をコンテナに登録してSwap

### スコープ管理
- singleton: アプリ起動中に1インスタンス（DB接続、外部APIクライアント）
- scoped: リクエストごとに1インスタンス（ユーザーコンテキスト付きサービス）
- transient: 毎回新規インスタンス（ステートレスなユーティリティ）

### ファクトリー登録
- 非同期初期化が必要なサービスはfactoryパターンで登録
- 循環依存はコンパイル時に検出（依存グラフを可視化）
```

---

## DIコンテナ実装の生成

```
DIコンテナを設計してください。

要件：
- singleton/scoped/transient スコープ
- コンストラクタインジェクション
- 非同期ファクトリー
- テスト時のモックSwap

生成ファイル: src/di/
```

---

## 生成されるDIコンテナ実装

```typescript
// src/di/container.ts — DIコンテナ

type Token<T> = symbol | string | (new (...args: any[]) => T);
type Scope = 'singleton' | 'transient';
type Factory<T> = (container: Container) => T | Promise<T>;

interface Registration<T> {
  factory: Factory<T>;
  scope: Scope;
  instance?: T; // singletonの場合のキャッシュ
}

export class Container {
  private readonly registry = new Map<Token<any>, Registration<any>>();
  private readonly resolving = new Set<Token<any>>(); // 循環依存検出

  // シングルトン登録
  singleton<T>(token: Token<T>, factory: Factory<T>): this {
    this.registry.set(token, { factory, scope: 'singleton' });
    return this;
  }

  // トランジェント登録（毎回新規生成）
  transient<T>(token: Token<T>, factory: Factory<T>): this {
    this.registry.set(token, { factory, scope: 'transient' });
    return this;
  }

  // インスタンスを直接登録（テスト用モック注入に便利）
  register<T>(token: Token<T>, instance: T): this {
    this.registry.set(token, { factory: () => instance, scope: 'singleton', instance });
    return this;
  }

  async resolve<T>(token: Token<T>): Promise<T> {
    const registration = this.registry.get(token);
    if (!registration) {
      throw new Error(`No registration found for token: ${String(token)}`);
    }

    // シングルトンはキャッシュを返す
    if (registration.scope === 'singleton' && registration.instance !== undefined) {
      return registration.instance as T;
    }

    // 循環依存検出
    if (this.resolving.has(token)) {
      throw new Error(`Circular dependency detected for: ${String(token)}`);
    }

    this.resolving.add(token);
    try {
      const instance = await registration.factory(this);
      if (registration.scope === 'singleton') {
        registration.instance = instance;
      }
      return instance as T;
    } finally {
      this.resolving.delete(token);
    }
  }

  // テスト用: 既存登録をモックで上書き
  override<T>(token: Token<T>, mock: T): this {
    return this.register(token, mock);
  }
}
```

```typescript
// src/di/tokens.ts — DIトークン定義（型安全）

export const TOKENS = {
  // インフラ層
  Database:    Symbol('Database'),
  Redis:       Symbol('Redis'),
  Logger:      Symbol('Logger'),
  EmailClient: Symbol('EmailClient'),
  S3Client:    Symbol('S3Client'),

  // サービス層
  UserService:    Symbol('UserService'),
  OrderService:   Symbol('OrderService'),
  PaymentService: Symbol('PaymentService'),
  NotifyService:  Symbol('NotifyService'),
} as const;

// src/di/setup.ts — コンテナ設定

export async function createContainer(): Promise<Container> {
  const container = new Container();

  // インフラ層（全てシングルトン）
  container.singleton(TOKENS.Database, () =>
    new PrismaClient({ log: ['error'] })
  );

  container.singleton(TOKENS.Redis, () =>
    new Redis(process.env.REDIS_URL!)
  );

  container.singleton(TOKENS.Logger, () =>
    pino({ level: process.env.LOG_LEVEL ?? 'info' })
  );

  container.singleton(TOKENS.EmailClient, (c) =>
    new SendgridClient(process.env.SENDGRID_API_KEY!, c.resolve(TOKENS.Logger))
  );

  // サービス層（依存関係を明示）
  container.singleton(TOKENS.UserService, async (c) => {
    return new UserService(
      await c.resolve(TOKENS.Database),
      await c.resolve(TOKENS.Redis),
      await c.resolve(TOKENS.Logger)
    );
  });

  container.singleton(TOKENS.OrderService, async (c) => {
    return new OrderService(
      await c.resolve(TOKENS.Database),
      await c.resolve(TOKENS.UserService),
      await c.resolve(TOKENS.PaymentService)
    );
  });

  container.singleton(TOKENS.PaymentService, async (c) =>
    new StripePaymentService(
      process.env.STRIPE_SECRET_KEY!,
      await c.resolve(TOKENS.Logger)
    )
  );

  return container;
}

// サービスクラス（コンストラクタインジェクション）
export class OrderService {
  constructor(
    private readonly db: PrismaClient,
    private readonly userService: UserService,
    private readonly paymentService: PaymentService,
  ) {}

  async createOrder(userId: string, items: OrderItem[]): Promise<Order> {
    const user = await this.userService.findById(userId);
    const payment = await this.paymentService.charge(user.stripeCustomerId, total);
    return this.db.order.create({ data: { userId, items, paymentId: payment.id } });
  }
}
```

```typescript
// src/di/testHelpers.ts — テスト用DIヘルパー

export function createTestContainer(): Container {
  const container = new Container();

  // モックDB（in-memoryオブジェクト）
  const mockDb = createMockPrismaClient();
  container.register(TOKENS.Database, mockDb);

  // モックRedis
  const mockRedis = createMockRedis();
  container.register(TOKENS.Redis, mockRedis);

  // モックEmailClient（メールを実際には送らない）
  container.register(TOKENS.EmailClient, {
    send: jest.fn().mockResolvedValue({ messageId: 'test-123' }),
  });

  // 実際のサービスロジックはそのまま使う（インフラだけモック）
  container.singleton(TOKENS.UserService, async (c) =>
    new UserService(
      await c.resolve(TOKENS.Database),
      await c.resolve(TOKENS.Redis),
      pino({ level: 'silent' })
    )
  );

  return container;
}

// テストでの使用例
describe('OrderService', () => {
  let container: Container;
  let orderService: OrderService;

  beforeEach(async () => {
    container = createTestContainer();

    // 特定のテストだけPaymentServiceをモック
    container.register(TOKENS.PaymentService, {
      charge: jest.fn().mockResolvedValue({ id: 'pay_test123' }),
    } as unknown as PaymentService);

    orderService = await container.resolve(TOKENS.OrderService);
  });

  it('should create order with payment', async () => {
    const order = await orderService.createOrder('user-1', [{ productId: 'p1', quantity: 1 }]);
    expect(order.paymentId).toBe('pay_test123');
  });
});
```

---

## まとめ

Claude CodeでDIコンテナを設計する：

1. **CLAUDE.md** にコンストラクタインジェクション必須・singleton/scoped/transientスコープ定義・テスト時はコンテナにモック登録を明記
2. **循環依存検出** で解決中のトークンをSetで追跡——循環が発生した瞬間に明示的なエラーで通知
3. **container.override()** でテスト時にモックをSwap——本番コードを変更せずにインフラ層（DB/Redis/外部API）だけ置き換え可能
4. **実サービスロジックはそのままテスト** ——モックするのはインフラだけ。UserServiceのロジックは本物を使い、DBだけモックすることで統合テストに近い品質を確保

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
