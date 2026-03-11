---
title: "Claude CodeでデコレーターパターンをTypeScriptに適用する：横断的関心事の分離・キャッシュ・リトライ・ロギングデコレーター"
emoji: "🎀"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture"]
published: true
published_at: "2026-03-21 17:00"
---

## はじめに

「キャッシュ・リトライ・ロギングを全てのサービスメソッドに書き直している」「横断的関心事がビジネスロジックに混在している」——デコレーターパターンで横断的関心事をラッパーとして分離し、インターフェースを変えずに機能を付加する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにデコレーターパターン設計ルールを書く

```markdown
## デコレーターパターン設計ルール

### 適用場面
- キャッシュ: 引数をキーにして結果をキャッシュ
- リトライ: メソッド呼び出しを自動リトライ
- ロギング: 呼び出し・完了・エラーを自動記録
- バリデーション: 引数の事前検証
- メトリクス: 実行時間の自動計測

### 2種類の実装
- クラスベース: インターフェースを実装してラップ（GoFパターン）
- TypeScript デコレーター: @Cached @Retry @Log（メタデータベース）

### 原則
- 元のインターフェースを変えない
- デコレーターは組み合わせ可能（キャッシュ + ロギング等）
- デコレーターはビジネスロジックを知らない
```

---

## デコレーターパターン実装の生成

```
デコレーターパターンを設計してください。

要件：
- キャッシュデコレーター（Redis）
- リトライデコレーター
- ロギングデコレーター
- TypeScript デコレーター構文

生成ファイル: src/shared/decorators/
```

---

## 生成されるデコレーターパターン実装

```typescript
// src/shared/decorators/classDecorators.ts — クラスベースのデコレーター（GoFパターン）

// キャッシュデコレーター
export class CachedUserRepository implements IUserRepository {
  private readonly cache = new Map<string, { user: User; expiresAt: number }>();

  constructor(
    private readonly inner: IUserRepository,
    private readonly ttlMs: number = 60_000
  ) {}

  async findById(id: string): Promise<User | null> {
    const cached = this.cache.get(id);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.user;
    }

    const user = await this.inner.findById(id);
    if (user) {
      this.cache.set(id, { user, expiresAt: Date.now() + this.ttlMs });
    }
    return user;
  }

  async save(user: User): Promise<void> {
    await this.inner.save(user);
    this.cache.delete(user.id);  // キャッシュを無効化
  }

  async delete(id: string): Promise<void> {
    await this.inner.delete(id);
    this.cache.delete(id);
  }
}

// ロギングデコレーター
export class LoggingUserRepository implements IUserRepository {
  constructor(
    private readonly inner: IUserRepository,
    private readonly logger: Logger
  ) {}

  async findById(id: string): Promise<User | null> {
    const start = Date.now();
    try {
      const user = await this.inner.findById(id);
      this.logger.debug({ id, found: !!user, durationMs: Date.now() - start }, 'findById');
      return user;
    } catch (error) {
      this.logger.error({ id, error, durationMs: Date.now() - start }, 'findById failed');
      throw error;
    }
  }

  async save(user: User): Promise<void> {
    const start = Date.now();
    await this.inner.save(user);
    this.logger.info({ userId: user.id, durationMs: Date.now() - start }, 'User saved');
  }

  async delete(id: string): Promise<void> {
    await this.inner.delete(id);
    this.logger.info({ id }, 'User deleted');
  }
}

// デコレーターを重ねる（キャッシュ + ロギング）
const userRepository = new CachedUserRepository(
  new LoggingUserRepository(
    new PrismaUserRepository(db),
    logger
  ),
  60_000
);
```

```typescript
// src/shared/decorators/methodDecorators.ts — TypeScriptデコレーター構文

// Cacheデコレーター
export function Cache(options: {
  ttlMs: number;
  keyFn?: (...args: any[]) => string;
}): MethodDecorator {
  return (target, propertyKey, descriptor) => {
    const originalMethod = descriptor.value as Function;
    const cache = new Map<string, { value: unknown; expiresAt: number }>();

    (descriptor as any).value = async function (...args: any[]) {
      const key = options.keyFn
        ? options.keyFn(...args)
        : `${String(propertyKey)}_${JSON.stringify(args)}`;

      const cached = cache.get(key);
      if (cached && cached.expiresAt > Date.now()) {
        return cached.value;
      }

      const result = await originalMethod.apply(this, args);
      cache.set(key, { value: result, expiresAt: Date.now() + options.ttlMs });
      return result;
    };

    return descriptor;
  };
}

// Retryデコレーター
export function Retry(options: {
  maxAttempts: number;
  baseDelayMs: number;
  retryIf?: (error: Error) => boolean;
}): MethodDecorator {
  return (target, propertyKey, descriptor) => {
    const originalMethod = descriptor.value as Function;

    (descriptor as any).value = async function (...args: any[]) {
      let lastError: Error;

      for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error as Error;

          if (
            attempt === options.maxAttempts ||
            (options.retryIf && !options.retryIf(lastError))
          ) {
            throw lastError;
          }

          const delay = options.baseDelayMs * Math.pow(2, attempt - 1);
          await new Promise(resolve => setTimeout(resolve, delay * (0.5 + Math.random() * 0.5)));
        }
      }

      throw lastError!;
    };

    return descriptor;
  };
}

// Measureデコレーター（実行時間計測）
export function Measure(metricName?: string): MethodDecorator {
  return (target, propertyKey, descriptor) => {
    const originalMethod = descriptor.value as Function;
    const name = metricName ?? `${target.constructor.name}.${String(propertyKey)}`;

    (descriptor as any).value = async function (...args: any[]) {
      const start = performance.now();
      try {
        const result = await originalMethod.apply(this, args);
        const durationMs = performance.now() - start;
        if (durationMs > 100) {
          logger.warn({ name, durationMs }, 'Slow method execution');
        }
        operationHistogram.observe({ operation: name }, durationMs);
        return result;
      } catch (error) {
        errorCounter.inc({ operation: name });
        throw error;
      }
    };

    return descriptor;
  };
}

// 使用例（デコレーターを重ねる）
export class ProductService {
  constructor(private readonly productRepo: IProductRepository) {}

  @Cache({ ttlMs: 60_000, keyFn: (id: string) => `product:${id}` })
  @Measure('ProductService.findById')
  async findById(id: string): Promise<Product | null> {
    return this.productRepo.findById(id);
  }

  @Cache({ ttlMs: 30_000, keyFn: (...args) => `products:featured:${JSON.stringify(args)}` })
  @Measure('ProductService.getFeatured')
  async getFeatured(category: string, limit: number): Promise<Product[]> {
    return this.productRepo.findFeatured(category, limit);
  }

  @Retry({ maxAttempts: 3, baseDelayMs: 500, retryIf: isNetworkError })
  @Measure('ProductService.syncWithExternalCatalog')
  async syncWithExternalCatalog(): Promise<void> {
    // 外部APIとの同期（ネットワークエラーをリトライ）
    const products = await this.externalApi.getLatestCatalog();
    await this.productRepo.bulkUpsert(products);
  }
}
```

---

## まとめ

Claude Codeでデコレーターパターンを設計する：

1. **CLAUDE.md** にクラスベースデコレーターはインターフェースをラップ・TypeScriptデコレーターはメソッドをアスペクトで強化・デコレーターは組み合わせ可能・ビジネスロジックを知らないを明記
2. **クラスベースデコレーター（GoFパターン）** ——`CachedUserRepository(LoggingUserRepository(PrismaUserRepository()))`と重ねるだけ。Prismaを直接使うコードはキャッシュもロギングも知らない
3. **TypeScriptデコレーター（`@Cache`・`@Retry`・`@Measure`）** ——`@Cache({ ttlMs: 60_000 })`をメソッドに付けるだけでキャッシュが動く。ビジネスロジックの実装に1行も変更不要
4. **デコレーターの重ね合わせ** ——`@Cache` + `@Measure` + `@Retry`を同時に付与できる。適用順は下から上（まずRetry、次にMeasure、最後にCache）。組み合わせを変えるだけで異なる動作を実現

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
