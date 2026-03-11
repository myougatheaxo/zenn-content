---
title: "Claude CodeでTypeScript高度な型を設計する：Utility Types・Conditional Types・型安全API"
emoji: "🔷"
type: "tech"
topics: ["claudecode", "typescript", "nodejs", "type-safety"]
published: true
---

## はじめに

`any`を使うとTypeScriptの恩恵が消える。型推論を活かしたコードは自己文書化されて、リファクタリング時のバグを大幅に減らせる。Claude Codeに高度な型設計を生成させる。

---

## CLAUDE.mdにTypeScript型ルールを書く

```markdown
## TypeScript型設計ルール

### 禁止
- any禁止（unknown + 型ガードを使う）
- as T の型アサーション禁止（型ガード関数を使う）
- ts-ignore / @ts-expect-error 禁止（型エラーは型を直す）

### 推奨パターン
- APIレスポンスはZodでparseして型を付ける
- Discriminated Union で状態を表現
- Template Literal Typesでイベント名・キー名を型安全に
- satisfies演算子で型を絞りながら推論を保持

### 設定
- strict: true 必須
- noUncheckedIndexedAccess: true（配列アクセスをundefinedを含む型に）
- exactOptionalPropertyTypes: true（undefined代入と省略を区別）
```

---

## 高度な型パターンの生成

```
以下の型安全パターンをTypeScriptで設計してください：

1. Discriminated Union でAPIレスポンスの状態管理
2. Template Literal Types でイベント名を型安全に
3. satisfies演算子でルート定義
4. unknown + 型ガードでAny排除
```

---

## Discriminated Union（状態管理）

```typescript
// APIレスポンスの状態をDiscriminated Unionで表現
type ApiState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error; retryCount: number };

function renderUserList(state: ApiState<User[]>) {
  switch (state.status) {
    case 'idle': return <div>Click to load</div>;
    case 'loading': return <Spinner />;
    case 'success': return state.data.map(u => <UserCard key={u.id} user={u} />);
    case 'error': return <ErrorMessage error={state.error} retry={state.retryCount} />;
  }
}
```

---

## Template Literal Types（イベント名）

```typescript
// DBエンティティのCRUDイベントを型安全に定義
type Entity = 'user' | 'order' | 'product';
type Action = 'created' | 'updated' | 'deleted';
type EventName = `${Entity}.${Action}`;
// → 'user.created' | 'user.updated' | ... | 'product.deleted' (9通り)

// イベントペイロードの型マッピング
type EventPayloads = {
  'user.created': { user: User };
  'user.updated': { user: User; changes: Partial<User> };
  'order.created': { order: Order; user: User };
  // ...
};

// 型安全なイベントエミッター
class TypedEventEmitter {
  emit<K extends keyof EventPayloads>(event: K, payload: EventPayloads[K]): void {
    // 実装
  }

  on<K extends keyof EventPayloads>(
    event: K,
    handler: (payload: EventPayloads[K]) => void
  ): void {
    // 実装
  }
}

// 使用例: 型エラーはコンパイル時に検出
emitter.emit('user.created', { user }); // OK
emitter.emit('user.created', { order }); // Type error!
```

---

## satisfies演算子（ルート定義）

```typescript
// satisfiesで型を検証しながら推論を保持
const routes = {
  home: { path: '/', title: 'Home', auth: false },
  dashboard: { path: '/dashboard', title: 'Dashboard', auth: true },
  profile: { path: '/profile/:id', title: 'Profile', auth: true },
} satisfies Record<string, { path: string; title: string; auth: boolean }>;

// 型チェックは通るが推論は保持（routes.home.path の型は '/' ではなく string）
type RouteName = keyof typeof routes; // 'home' | 'dashboard' | 'profile'
```

---

## unknown + 型ガード（any排除）

```typescript
// anyの代わりにunknown + 型ガード
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof (value as User).id === 'string' &&
    'email' in value &&
    typeof (value as User).email === 'string'
  );
}

// 外部APIレスポンスを安全に扱う
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();

  if (!isUser(data)) {
    throw new Error(`Invalid user response: ${JSON.stringify(data)}`);
  }

  return data; // ここでは User 型
}
```

---

## Utility Types の活用

```typescript
// 読み取り専用DTO
type UserDTO = Readonly<Pick<User, 'id' | 'name' | 'email' | 'createdAt'>>;

// 更新用（全フィールド任意、idは除外）
type UpdateUserInput = Partial<Omit<User, 'id' | 'createdAt' | 'updatedAt'>>;

// APIエラーレスポンス
type ApiError = {
  code: string;
  message: string;
  details?: Record<string, string[]>;
};

// 非同期関数の戻り値の型を取得
type UserServiceReturn = Awaited<ReturnType<typeof getUserProfile>>;
```

---

## まとめ

Claude CodeでTypeScript高度な型を設計する：

1. **CLAUDE.md** にany禁止・型アサーション禁止・strict:trueを明記
2. **Discriminated Union** で状態管理を型安全に
3. **Template Literal Types** でイベント名・キー名を型で制約
4. **unknown + 型ガード** でexternal dataを安全に扱う

---

*TypeScriptの型設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
