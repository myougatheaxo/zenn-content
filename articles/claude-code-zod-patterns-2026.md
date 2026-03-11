---
title: "Claude CodeでZodバリデーションパターンを設計する：スキーマ再利用と型推論"
emoji: "✅"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "zod", "security"]
published: true
---

## はじめに

バリデーションを実装しないと不正なデータがDBに入る。文字列のつもりがnullになる。Claude CodeにZodを使った完全バリデーションを生成させる。

---

## CLAUDE.mdにバリデーションルールを書く

```markdown
## 入力バリデーションルール

### 必須事項
- 全てのAPIエンドポイントにZodバリデーションを設定
- バリデーションはルートハンドラーの最初で実行
- バリデーションエラーは422を返す

### Zodスキーマ設計
- 文字列: 最大長を必ず設定（.max(N)）
- メール: .email()
- 数値: .int() or .finite()
- 未知フィールドは .strict() で拒否

### エラーレスポンス形式
{ "errors": [{ "field": "email", "message": "Invalid email format" }] }
```

---

## バリデーション設計の生成

```
Zodを使ったAPIバリデーションを設計してください。

エンドポイント: POST /api/users
バリデーション:
- name: 1-100文字
- email: メール形式
- age: 0-150の整数（任意）
- role: "user" | "admin" | "moderator"

要件：
- バリデーションエラーは422でフィールド名付き返却
- ミドルウェアで再利用可能な形式
- TypeScriptの型推論を活かす

生成ファイル:
- src/schemas/userSchemas.ts
- src/middleware/validate.ts
```

---

## 生成されるZodスキーマ

```typescript
// src/schemas/userSchemas.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string()
    .min(1, 'Name is required')
    .max(100, 'Name must be 100 characters or less')
    .trim(),
  email: z.string()
    .email('Invalid email format')
    .max(255, 'Email too long')
    .toLowerCase(),
  age: z.number()
    .int('Age must be an integer')
    .min(0).max(150)
    .optional(),
  role: z.enum(['user', 'admin', 'moderator']).default('user'),
  profileUrl: z.string().url('Invalid URL format').optional(),
}).strict(); // 余分なフィールドはエラー

// updateはcreateのPartial（全フィールド任意）
export const updateUserSchema = createUserSchema.partial().omit({ email: true });

// TypeScriptの型はスキーマから自動生成
export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
```

---

## バリデーションミドルウェア

```typescript
// src/middleware/validate.ts
import { ZodSchema } from 'zod';

export function validate<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      const errors = result.error.errors.map(err => ({
        field: err.path.join('.'),
        message: err.message,
      }));
      res.status(422).json({ errors });
      return;
    }

    // バリデーション済みのデータでreq.bodyを上書き（型安全）
    req.body = result.data;
    next();
  };
}

export function validateQuery<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.query);

    if (!result.success) {
      const errors = result.error.errors.map(err => ({
        field: err.path.join('.'),
        message: err.message,
      }));
      res.status(422).json({ errors });
      return;
    }

    req.query = result.data as any;
    next();
  };
}
```

```typescript
// src/routes/users.ts
router.post('/users', authenticate, validate(createUserSchema), async (req, res) => {
  const input = req.body as CreateUserInput; // 型安全
  const user = await createUser(input);
  res.status(201).json(user);
});
```

---

## クエリパラメータのバリデーション

```typescript
const paginationSchema = z.object({
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.string().optional(),
  sort: z.enum(['createdAt', 'updatedAt', 'name']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

router.get('/users', authenticate, validateQuery(paginationSchema), async (req, res) => {
  const { limit, cursor, sort, order } = req.query as z.infer<typeof paginationSchema>;
  // ...
});
```

---

## ネストしたオブジェクト・配列

```typescript
export const createOrderSchema = z.object({
  customerId: z.string().uuid('Invalid customer ID'),
  items: z.array(
    z.object({
      productId: z.string().uuid(),
      quantity: z.number().int().positive().max(999),
    })
  ).min(1, 'Order must have at least one item').max(100),
  shippingAddress: z.object({
    street: z.string().min(1).max(200),
    city: z.string().min(1).max(100),
    postalCode: z.string().regex(/^\d{3}-\d{4}$/, 'Invalid postal code'),
  }),
});
```

---

## まとめ

Claude CodeでZodバリデーションを設計する：

1. **CLAUDE.md** に全エンドポイントでバリデーション必須・.strict()で余分フィールド拒否を明記
2. **safeParse** でランタイムエラーを防ぎ、422+フィールド情報でエラー返却
3. **バリデーション済みデータでreq.bodyを上書き** してルートハンドラー内で型安全に使う
4. **z.infer** でTypeScriptの型をスキーマから自動生成

---

*バリデーション漏れを検出するスキルは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
