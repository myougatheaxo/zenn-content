---
title: "Claude CodeでTypeScriptの型安全性を最大化する：Discriminated Union・Branded Types"
emoji: "🔷"
type: "tech"
topics: ["claudecode", "typescript", "型安全", "nodejs"]
published: true
---

## はじめに: anyを使わないTypeScriptとClaude Code

TypeScriptを書いていると、つい`any`で逃げたくなる瞬間があります。APIレスポンスの型が複雑だったり、既存のJSコードをマイグレーション中だったり。しかし`any`は型チェックを無効化し、バグを実行時まで隠蔽します。

Claude Codeを活用すれば、`any`を一切使わない型安全なコードをプロンプト一発で生成できます。本記事では、CLAUDE.mdに型安全ルールを定義してから、実践的なTypeScriptパターンをClaude Codeに生成させる手法を解説します。

---

## CLAUDE.mdにTypeScript厳格ルールを書く

Claude CodeはプロジェクトルートのCLAUDE.mdをコンテキストとして読み込みます。ここにTypeScriptの制約を書くことで、生成コードの品質を一貫して高く保てます。

```markdown
## TypeScript 厳格ルール

- `tsconfig.json` に `"strict": true` を必須とする
- `any` 型の使用を禁止。代わりに `unknown` を使い、型ガードで絞り込む
- `as` による型アサーションは禁止。型ガード関数を実装すること
- `// @ts-ignore` および `// @ts-expect-error` は使用禁止
- すべての関数に戻り値の型アノテーションを明示する
- `null` チェックは必ず行う（strictNullChecks: true）
- 外部APIのレスポンスには必ず `unknown` を使い、zodまたは型ガードで検証する
```

この設定をCLAUDE.mdに追加するだけで、以後のコード生成はすべてこのルールに従います。

---

## Discriminated Unionパターンの生成プロンプト

Discriminated Union（判別可能なユニオン型）は、TypeScriptで最も強力なパターンの一つです。特にエラーハンドリングで`try/catch`と`any`の組み合わせを排除できます。

### プロンプト例

```
CLAUDE.mdのTypeScriptルールに従い、以下を実装してください。

Result<T, E>型を使った型安全なエラーハンドリングパターン：
- ok: true の場合は data: T を持つ
- ok: false の場合は error: E を持つ
- unwrapOrThrow(), mapResult(), flatMapResult() ヘルパーを追加
- async版のAsyncResult<T, E>も実装
- anyは一切使わないこと

使用例：
async function fetchUser(id: string): AsyncResult<User, ApiError>
```

### 生成されるコード例

```typescript
type Result<T, E> =
  | { ok: true; data: T }
  | { ok: false; error: E };

type AsyncResult<T, E> = Promise<Result<T, E>>;

function ok<T>(data: T): Result<T, never> {
  return { ok: true, data };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

function unwrapOrThrow<T, E>(result: Result<T, E>): T {
  if (result.ok) return result.data;
  throw result.error;
}

function mapResult<T, U, E>(
  result: Result<T, E>,
  fn: (data: T) => U
): Result<U, E> {
  if (result.ok) return ok(fn(result.data));
  return result;
}

// 使用例
type ApiError =
  | { type: "NOT_FOUND"; id: string }
  | { type: "UNAUTHORIZED" }
  | { type: "NETWORK_ERROR"; message: string };

async function fetchUser(id: string): AsyncResult<User, ApiError> {
  try {
    const res = await fetch(`/api/users/${encodeURIComponent(id)}`);
    if (res.status === 404) return err({ type: "NOT_FOUND", id });
    if (res.status === 401) return err({ type: "UNAUTHORIZED" });
    if (!res.ok) return err({ type: "NETWORK_ERROR", message: res.statusText });
    const raw: unknown = await res.json();
    return ok(parseUser(raw)); // zodや型ガードで検証
  } catch (e) {
    const message = e instanceof Error ? e.message : "unknown";
    return err({ type: "NETWORK_ERROR", message });
  }
}
```

`try/catch`で`any`を受け取る代わりに、すべてのエラーケースが型として表現されています。コンパイル時に網羅チェックが効くため、新しいエラーケースを追加した際に対処漏れがビルドエラーとして検出されます。

---

## Branded Typesの生成プロンプト

`UserId`と`OrderId`はどちらも`string`ですが、誤って混用するとバグが発生します。Branded Typesで型レベルの区別を作ります。

### プロンプト例

```
CLAUDE.mdのTypeScriptルールに従い、Branded Types（Opaque Types）パターンを実装してください。

要件：
- UserId, OrderId, ProductId, Email の各Branded Typeを定義
- Brand<T, B> ユーティリティ型を使う
- 各型のバリデーション付きコンストラクタ関数を実装（parse～という命名）
- anyもasも使わないこと
- バリデーション失敗時はResult型でエラーを返すこと
```

### 生成されるコード例

```typescript
declare const __brand: unique symbol;
type Brand<T, B> = T & { readonly [__brand]: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type Email = Brand<string, "Email">;

type ValidationError = { field: string; message: string };

function parseUserId(raw: string): Result<UserId, ValidationError> {
  if (!raw || raw.length < 1) {
    return err({ field: "userId", message: "UserIdは1文字以上必要です" });
  }
  // UUID形式チェック等
  return ok(raw as UserId); // ここだけブランド付与のためasを許容（境界層）
}

function parseEmail(raw: string): Result<Email, ValidationError> {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(raw)) {
    return err({ field: "email", message: "メールアドレスの形式が不正です" });
  }
  return ok(raw as Email);
}

// 型の混同を防ぐ
function sendOrderConfirmation(userId: UserId, orderId: OrderId): void {
  // userId と orderId を混同するとコンパイルエラー
  console.log(`User ${userId} の注文 ${orderId} を確認しました`);
}

// これはコンパイルエラーになる
// sendOrderConfirmation(orderId, userId); // 引数の順序を間違えてもビルドが通らない
```

Branded Typesにより、同じ`string`型でも関数の引数で混用するとコンパイルエラーになります。実行時エラーではなくビルド時に検出できるのが重要です。

---

## Mapped Typesの活用プロンプト

OpenAPIスペックやREST APIのレスポンスから型を手動で書くのは非効率です。Mapped Typesで既存の型から派生型を自動生成させます。

### プロンプト例

```
CLAUDE.mdのTypeScriptルールに従い、以下のMapped Typesユーティリティを実装してください。

1. APIレスポンス型からフロントエンド向けCamelCase型を生成するSnakeToCamelCase変換
2. Partial<T>の深いネスト版DeepPartial<T>
3. Readonly<T>の深いネスト版DeepReadonly<T>
4. 型のキーをReadonlyにしてnullableを除去するStrictRequired<T>
5. anyもasも使わないこと（型レベルの操作のみ）
```

### 生成されるコード例

```typescript
// SnakeToCamelCase: 型レベルの文字列変換
type SnakeToCamelCase<S extends string> =
  S extends `${infer Head}_${infer Tail}`
    ? `${Head}${Capitalize<SnakeToCamelCase<Tail>>}`
    : S;

type SnakeKeysToCamelCase<T> = T extends object
  ? {
      [K in keyof T as SnakeToCamelCase<K & string>]: T[K] extends object
        ? SnakeKeysToCamelCase<T[K]>
        : T[K];
    }
  : T;

// APIレスポンスの例
type ApiUser = {
  user_id: string;
  created_at: string;
  profile_image_url: string | null;
};

// 自動的にCamelCaseに変換
type FrontendUser = SnakeKeysToCamelCase<ApiUser>;
// { userId: string; createdAt: string; profileImageUrl: string | null }

// DeepReadonly: イミュータブルなデータ構造
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

// StrictRequired: undefinedとnullを除去
type StrictRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};
```

これらの型ユーティリティを一度定義すれば、APIのスキーマ変更があっても型を手動で書き直す必要がありません。

---

## まとめ

| パターン | 解決する問題 |
|---------|------------|
| Discriminated Union + Result型 | `try/catch`での`any`排除、エラーの型安全な網羅 |
| Branded Types | 同一プリミティブ型の混用バグを型レベルで防止 |
| Mapped Types | APIレスポンスから派生型を自動生成、手動書き直し不要 |

CLAUDE.mdに厳格ルールを定義することで、これらのパターンを毎回プロンプトで説明しなくても一貫したコード品質を維持できます。`any`を使わないTypeScriptは、最初は難しく感じますが、Claude Codeのサポートがあれば実践に移すハードルは大幅に下がります。

---

**Code Review Pack（¥980）** の `/code-review` コマンドで、`any`使用箇所・型アサーション・unsafe操作を自動検出できます。
👉 https://prompt-works.jp
