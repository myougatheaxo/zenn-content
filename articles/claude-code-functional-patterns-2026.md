---
title: "Claude Codeで関数型パターンをTypeScriptに適用する：Either・Option・pipe・Result型で副作用を隔離"
emoji: "λ"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture"]
published: true
published_at: "2026-03-21 09:00"
---

## はじめに

「try-catchが散在してエラーの伝播が追えない」「null/undefinedチェックがネストして読めない」——Either型・Option型・Result型でエラーと欠損値を型で表現し、副作用を隔離する関数型パターンをClaude Codeに生成させる。

---

## CLAUDE.mdに関数型パターン設計ルールを書く

```markdown
## 関数型パターン設計ルール

### Result型の使用
- 予期されるエラー（バリデーション、NotFound）はResultで返す
- 予期外エラー（バグ）はthrowで伝播
- Result<T, E>: Ok(value) | Err(error) の2値型

### pipeとflow
- pipe: 値を関数チェーンで変換 pipe(value, fn1, fn2, fn3)
- 中間で失敗する可能性がある変換はResultをpipeで連鎖

### 副作用の隔離
- 純粋関数（入力→出力のみ、副作用なし）でビジネスロジックを書く
- IOアクション（DB・API）は端に追い出す
- テストで副作用なしに検証できる
```

---

## 関数型パターン実装の生成

```
TypeScript向け関数型パターンを設計してください。

要件：
- Result型（Ok/Err）の実装
- Option型（Some/None）の実装
- pipe関数
- ドメインロジックへの適用

生成ファイル: src/shared/fp/
```

---

## 生成される関数型パターン実装

```typescript
// src/shared/fp/result.ts — Result型

export type Result<T, E extends Error = Error> = Ok<T> | Err<E>;

export class Ok<T> {
  readonly _tag = 'Ok' as const;
  constructor(readonly value: T) {}

  isOk(): this is Ok<T> { return true; }
  isErr(): this is never { return false; }

  map<U>(fn: (value: T) => U): Result<U, never> {
    return ok(fn(this.value));
  }

  flatMap<U, E extends Error>(fn: (value: T) => Result<U, E>): Result<U, E> {
    return fn(this.value);
  }

  mapErr<F extends Error>(_fn: (error: never) => F): Result<T, F> {
    return this as unknown as Result<T, F>;
  }

  getOrElse(_fallback: T): T { return this.value; }
  getOrThrow(): T { return this.value; }
}

export class Err<E extends Error> {
  readonly _tag = 'Err' as const;
  constructor(readonly error: E) {}

  isOk(): this is never { return false; }
  isErr(): this is Err<E> { return true; }

  map<U>(_fn: (value: never) => U): Result<U, E> {
    return this as unknown as Result<U, E>;
  }

  flatMap<U>(_fn: (value: never) => Result<U, E>): Result<U, E> {
    return this as unknown as Result<U, E>;
  }

  mapErr<F extends Error>(fn: (error: E) => F): Result<never, F> {
    return err(fn(this.error));
  }

  getOrElse<T>(fallback: T): T { return fallback; }
  getOrThrow(): never { throw this.error; }
}

export const ok = <T>(value: T): Ok<T> => new Ok(value);
export const err = <E extends Error>(error: E): Err<E> => new Err(error);

// Result型ユーティリティ
export function fromThrowable<T, E extends Error>(
  fn: () => T,
  mapError: (e: unknown) => E
): Result<T, E> {
  try {
    return ok(fn());
  } catch (error) {
    return err(mapError(error));
  }
}

export async function fromPromise<T, E extends Error>(
  promise: Promise<T>,
  mapError: (e: unknown) => E
): Promise<Result<T, E>> {
  try {
    return ok(await promise);
  } catch (error) {
    return err(mapError(error));
  }
}

// 複数のResultを一括処理
export function collect<T, E extends Error>(results: Result<T, E>[]): Result<T[], E> {
  const values: T[] = [];
  for (const result of results) {
    if (result.isErr()) return result;
    values.push(result.value);
  }
  return ok(values);
}
```

```typescript
// src/shared/fp/option.ts — Option型

export type Option<T> = Some<T> | None;

export class Some<T> {
  readonly _tag = 'Some' as const;
  constructor(readonly value: T) {}

  isSome(): this is Some<T> { return true; }
  isNone(): this is never { return false; }

  map<U>(fn: (value: T) => U): Option<U> { return some(fn(this.value)); }
  flatMap<U>(fn: (value: T) => Option<U>): Option<U> { return fn(this.value); }
  getOrElse(_fallback: T): T { return this.value; }
  toResult<E extends Error>(error: E): Result<T, E> { return ok(this.value); }
}

export class None {
  readonly _tag = 'None' as const;
  isSome(): this is never { return false; }
  isNone(): this is None { return true; }

  map<U>(_fn: (value: never) => U): Option<U> { return none; }
  flatMap<U>(_fn: (value: never) => Option<U>): Option<U> { return none; }
  getOrElse<T>(fallback: T): T { return fallback; }
  toResult<T, E extends Error>(error: E): Result<T, E> { return err(error); }
}

export const some = <T>(value: T): Some<T> => new Some(value);
export const none: None = new None();
export const fromNullable = <T>(value: T | null | undefined): Option<T> =>
  value != null ? some(value) : none;
```

```typescript
// src/shared/fp/pipe.ts — pipe関数

// TypeScriptの型推論で連鎖する関数の型を解決
export function pipe<A>(a: A): A;
export function pipe<A, B>(a: A, ab: (a: A) => B): B;
export function pipe<A, B, C>(a: A, ab: (a: A) => B, bc: (b: B) => C): C;
export function pipe<A, B, C, D>(a: A, ab: (a: A) => B, bc: (b: B) => C, cd: (c: C) => D): D;
export function pipe<A, B, C, D, E>(
  a: A, ab: (a: A) => B, bc: (b: B) => C, cd: (c: C) => D, de: (d: D) => E
): E;
export function pipe(value: unknown, ...fns: Array<(x: unknown) => unknown>): unknown {
  return fns.reduce((acc, fn) => fn(acc), value);
}

// ドメインロジックへの適用例

// バリデーションとビジネスルールをResult/pipeで連鎖
interface CreateOrderInput {
  userId: string;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
}

function validateUserId(input: CreateOrderInput): Result<CreateOrderInput, ValidationError> {
  if (!input.userId || input.userId.length < 5) {
    return err(new ValidationError([{ field: 'userId', message: 'Invalid user ID' }]));
  }
  return ok(input);
}

function validateItems(input: CreateOrderInput): Result<CreateOrderInput, ValidationError> {
  if (input.items.length === 0) {
    return err(new ValidationError([{ field: 'items', message: 'At least one item required' }]));
  }
  const invalidItems = input.items.filter(i => i.quantity <= 0 || i.unitPrice <= 0);
  if (invalidItems.length > 0) {
    return err(new ValidationError(invalidItems.map((_, i) => ({
      field: `items[${i}]`,
      message: 'Quantity and unitPrice must be positive',
    }))));
  }
  return ok(input);
}

function validateMaxAmount(maxAmount: number) {
  return (input: CreateOrderInput): Result<CreateOrderInput, DomainError> => {
    const total = input.items.reduce((sum, i) => sum + i.quantity * i.unitPrice, 0);
    if (total > maxAmount) {
      return err(new DomainError(`Order total ${total} exceeds maximum ${maxAmount}`));
    }
    return ok(input);
  };
}

function toOrder(input: CreateOrderInput): Result<Order, DomainError> {
  return fromThrowable(
    () => Order.create(input.userId, input.items.map(i => ({
      productId: i.productId,
      quantity: i.quantity,
      price: Money.of(i.unitPrice, 'JPY'),
    }))),
    (e) => new DomainError(String(e))
  );
}

// pipeで全バリデーションを連鎖（早期リターン）
export async function createOrderWithValidation(
  input: CreateOrderInput
): Promise<Result<Order, ValidationError | DomainError>> {
  const result = pipe(
    ok<CreateOrderInput>(input),
    r => r.flatMap(validateUserId),
    r => r.flatMap(validateItems),
    r => r.flatMap(validateMaxAmount(500_000)),
    r => r.flatMap(toOrder)
  );

  return result;
}

// 使用例（ユースケース層）
const result = await createOrderWithValidation({
  userId: req.user.id,
  items: req.body.items,
});

if (result.isErr()) {
  if (result.error instanceof ValidationError) {
    return res.status(400).json({ code: 'VALIDATION_ERROR', ...result.error });
  }
  return res.status(422).json({ code: 'DOMAIN_ERROR', message: result.error.message });
}

const order = result.value;
await orderRepository.save(order);
res.status(201).json({ orderId: order.id });
```

---

## まとめ

Claude Codeで関数型パターンをTypeScriptに適用する：

1. **CLAUDE.md** に予期されるエラーはResult型で返す・予期外エラーはthrow・Option型でnull/undefinedを型で表現・pipeで変換を連鎖させる を明記
2. **Result型（Ok/Err）** でtry-catchを不要に——`validateUserId()`が`Result<Input, ValidationError>`を返すことで、呼び出し側は型システムでエラーの存在を強制される。忘れてcatchしないことが型エラーになる
3. **`pipe(ok(input), r => r.flatMap(validate1), r => r.flatMap(validate2))`** で早期リターンを型安全に実現——最初のErrで以降の処理がスキップされ、最後まで成功したものだけOkになる。ネストしたif-elseチェーンを平坦化
4. **`fromThrowable()`** でthrowするライブラリをResultに変換——Prismaや外部ライブラリがthrowしてくる例外をResult型に持ち込める。ドメインロジック内でtry-catchを書かなくてよくなる

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
