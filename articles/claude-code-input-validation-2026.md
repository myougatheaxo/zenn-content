---
title: "Claude Codeでバリデーションを統一する：ZodでAPIの入口を守る"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "zod", "typescript", "nodejs", "バリデーション"]
published: true
---

## はじめに

バリデーション不足は、地味だが深刻なセキュリティリスクだ。

- 型チェックなしでDBに渡したSQLインジェクション
- 数値フィールドに文字列が入ってサーバーがクラッシュ
- メールアドレス未検証でスパム踏み台になる

Claude Codeを制約なしで使うと、こういった問題が混入しやすい。`typeof req.body.age === 'number'` のような手書きifチェックや、検証なしでそのまま処理するコードを生成してしまう。

**CLAUDE.mdにZodを標準として明示することで、APIの入口を統一できる。**

---

## CLAUDE.mdにバリデーションルールを書く

```markdown
## バリデーション規約

### 標準ライブラリ
- バリデーションはZodを使う（他のライブラリ禁止）
- 手書きのifチェック（typeof/instanceof等）は禁止
- Zodスキーマは `src/schemas/` に配置し、ルートハンドラから分離する

### スキーマ設計
- APIリクエスト（body/query/params）は必ずZodでパースする
- `z.parse()` ではなく `z.safeParse()` を使い、エラーを安全に捕捉する
- `.strict()` を使ってスキーマ外のフィールドを拒否する
- 文字列フィールドは `.trim()` と `.min(1)` を組み合わせて空白文字を弾く

### エラーレスポンス
- バリデーションエラーは HTTP 422 Unprocessable Entity で返す
- エラーボディは `{ errors: ZodIssue[] }` 形式で統一する
- フィールド名とエラーメッセージを必ず含める（フロントエンドが表示できるよう）

### ファイル構成
- `src/schemas/user.schema.ts` — ユーザー関連スキーマ
- `src/schemas/product.schema.ts` — プロダクト関連スキーマ
- `src/middleware/validate.ts` — 共通バリデーションミドルウェア
```

このルールをCLAUDE.mdに書いておくと、Claude Codeは新しいAPIを追加するたびに自動でZodスキーマを生成する。

---

## Zodスキーマの生成プロンプト例

### プロンプト

```
POST /api/products エンドポイントを追加して。
リクエストボディのZodスキーマを src/schemas/product.schema.ts に作成し、
バリデーションミドルウェア経由でルートに適用して。
```

### 生成されるコード例

**src/schemas/product.schema.ts**

```typescript
import { z } from "zod";

export const CreateProductSchema = z
  .object({
    name: z.string().trim().min(1, "商品名は必須です").max(100, "100文字以内で入力してください"),
    price: z.number().int().positive("価格は1以上の整数を指定してください"),
    category: z.enum(["electronics", "clothing", "food"], {
      errorMap: () => ({ message: "有効なカテゴリを選択してください" }),
    }),
    description: z.string().trim().max(500).optional(),
    tags: z.array(z.string().trim().min(1)).max(10).default([]),
  })
  .strict(); // 未知フィールドを拒否

export type CreateProductInput = z.infer<typeof CreateProductSchema>;
```

**src/middleware/validate.ts**

```typescript
import { Request, Response, NextFunction } from "express";
import { ZodSchema } from "zod";

export function validate(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(422).json({
        errors: result.error.issues.map((issue) => ({
          field: issue.path.join("."),
          message: issue.message,
        })),
      });
    }
    req.body = result.data; // パース済みデータで上書き（型安全）
    next();
  };
}
```

**src/routes/products.ts**

```typescript
import { Router } from "express";
import { validate } from "../middleware/validate";
import { CreateProductSchema } from "../schemas/product.schema";

const router = Router();

router.post("/", validate(CreateProductSchema), async (req, res) => {
  // req.body は CreateProductInput 型で保証済み
  const { name, price, category, description, tags } = req.body;
  // DB保存処理...
  res.status(201).json({ message: "作成しました" });
});

export default router;
```

---

## エラーレスポンスの統一プロンプト

バリデーションエラーの形式が散在すると、フロントエンドの実装が複雑になる。

### プロンプト

```
バリデーションエラーは全て 422 Unprocessable Entity で返して。
レスポンス形式は { errors: [{ field: string, message: string }] } に統一して。
ZodのIssueオブジェクトをそのまま返さないこと（内部情報を露出させないため）。
```

### レスポンス例

```json
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "errors": [
    { "field": "name", "message": "商品名は必須です" },
    { "field": "price", "message": "価格は1以上の整数を指定してください" }
  ]
}
```

クライアント側はフィールド名でエラーを特定でき、フォームの該当箇所にエラーを表示できる。

---

## Hooksでバリデーション漏れをチェック

Claude Codeが新しいルートを追加したとき、バリデーションを忘れていないかを自動検出する。

**`.claude/hooks/pre-tool-call.sh`**

```bash
#!/bin/bash
# Expressルートにバリデーションがない場合にブロックする

TOOL_NAME="$1"
FILE_PATH="$2"

# ルートファイルの変更のみチェック
if [[ "$TOOL_NAME" == "write_file" && "$FILE_PATH" == *"routes/"* ]]; then
  # stdin からファイル内容を読み込み
  CONTENT=$(cat)

  # router.post/put/patch があるがvalidateミドルウェアがない場合は警告
  if echo "$CONTENT" | grep -qE "router\.(post|put|patch)" && \
     ! echo "$CONTENT" | grep -q "validate("; then
    echo "WARNING: ルートにバリデーションミドルウェアがありません。" >&2
    echo "validate() を使って入力を検証してください。" >&2
    # ブロックする場合は exit 1
  fi
fi
```

このHookにより、バリデーションなしのルートをコミット前に検出できる。

---

## 実践的なスキーマパターン

よく使うパターンをまとめておく。

### クエリパラメータのバリデーション

```typescript
export const ListQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  search: z.string().trim().optional(),
});
```

`z.coerce` を使うと、クエリパラメータは文字列で届くため数値への変換を安全に行える。

### ネストしたオブジェクトのバリデーション

```typescript
export const CreateOrderSchema = z.object({
  items: z
    .array(
      z.object({
        productId: z.string().uuid(),
        quantity: z.number().int().min(1).max(99),
      })
    )
    .min(1, "1つ以上の商品を選択してください"),
  shippingAddress: z.object({
    postalCode: z.string().regex(/^\d{3}-\d{4}$/, "郵便番号の形式が正しくありません"),
    prefecture: z.string().min(1),
    city: z.string().min(1),
    street: z.string().min(1),
  }),
});
```

---

## まとめ

| やること | 効果 |
|---------|------|
| CLAUDE.mdにZodを標準指定 | Claude Codeが自動でスキーマ生成 |
| safeParse + 422レスポンス統一 | エラー処理の一貫性確保 |
| `src/schemas/` にスキーマ集約 | 再利用性と見通しの向上 |
| Hooksでバリデーション漏れ検出 | コミット前の自動チェック |

バリデーションは地味だが、APIの品質を左右する重要な仕組みだ。Claude Codeに任せるなら、まずルールを明示することが先決だ。

---

`/code-review` スキルがバリデーション漏れを自動検出。**Code Review Pack（¥980）** として PromptWorks で購入できます。https://prompt-works.jp
