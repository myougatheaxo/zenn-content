---
title: "Claude CodeでPrismaを使いこなす：型安全なDB操作とマイグレーション戦略"
emoji: "🗄️"
type: "tech"
topics: ["claudecode", "prisma", "typescript", "nodejs", "データベース"]
published: true
---

## はじめに

PrismaはTypeScriptとの相性が良いORMだが、N+1問題・トランザクション設計・マイグレーション戦略を間違えると本番で痛い目を見る。Claude Codeに設計方針を伝えて正しいパターンを生成させる。

---

## CLAUDE.mdにPrismaルールを書く

```markdown
## Prismaルール

### 基本設定
- Prismaクライアントはシングルトン: src/lib/prisma.ts で一元管理
- スキーマファイル: prisma/schema.prisma
- マイグレーションは必ず `prisma migrate deploy`（手動SQL禁止）

### クエリ設計
- N+1禁止: リレーションは include で取得（ループ内でクエリを叩かない）
- select を使って必要なフィールドだけ取得（SELECT * 禁止）
- 巨大テーブルのcountは避ける（_count を使うか、カウンターキャッシュを使う）

### トランザクション
- 複数テーブルにまたがる更新は必ずトランザクション
- `prisma.$transaction()` を使う
- タイムアウト: 10秒（デフォルト5秒）

### マイグレーション
- 本番環境でのロールバックは手動（事前に巻き戻し手順を書く）
- カラム削除は3フェーズで実施: 非推奨化 → コード修正 → 削除
- 外部キー制約の追加はダウンタイムが発生する（注意）

### セキュリティ
- ユーザー入力を直接クエリに渡さない（Prismaの型安全を信頼する）
- `prisma.$queryRaw` は必要最小限（SQLインジェクションリスク）
```

---

## Prismaシングルトンの生成

```
PrismaClientのシングルトンを生成してください。

要件：
- 開発環境: ホットリロード時のClient過多を防ぐ（global変数を利用）
- 本番環境: 通常のインスタンス
- ログ設定: クエリとエラーをログ記録（本番ではエラーのみ）
- 接続プール: DATABASE_URL に ?connection_limit=10 を含める

保存先: src/lib/prisma.ts
```

```typescript
// 生成される src/lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const prismaClientSingleton = () => {
  return new PrismaClient({
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'error', 'warn']
      : ['error'],
  });
};

type PrismaClientSingleton = ReturnType<typeof prismaClientSingleton>;

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClientSingleton | undefined;
};

export const prisma = globalForPrisma.prisma ?? prismaClientSingleton();

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

---

## N+1問題を防ぐクエリの生成

```
以下のコードのN+1問題を修正してください。

問題のあるコード:
const posts = await prisma.post.findMany();
// N+1: 各投稿ごとにAuthorとCommentsを別クエリで取得している
for (const post of posts) {
  post.author = await prisma.user.findUnique({ where: { id: post.authorId } });
  post.comments = await prisma.comment.findMany({ where: { postId: post.id } });
}

要件：
- include を使って1クエリで取得
- 必要なフィールドだけ select
- ページネーション: cursor-based（take + cursor）
```

```typescript
// 修正後
const posts = await prisma.post.findMany({
  select: {
    id: true,
    title: true,
    publishedAt: true,
    author: {
      select: { id: true, name: true },
    },
    comments: {
      select: { id: true, content: true, createdAt: true },
      orderBy: { createdAt: 'desc' },
      take: 5, // 最新5件のみ
    },
    _count: { select: { comments: true } }, // コメント総数はカウンターで
  },
  where: { publishedAt: { not: null } },
  orderBy: { publishedAt: 'desc' },
  take: 20,
  ...(cursor && { skip: 1, cursor: { id: cursor } }),
});
```

---

## トランザクションの生成

```
注文処理のトランザクションを生成してください。

操作:
1. Orderを作成
2. OrderItemsを一括作成
3. 各商品の在庫を減らす
4. ユーザーのポイントを使用する

要件：
- 全ての操作が成功しなければロールバック
- タイムアウト: 15秒
- エラー時はOrderを削除しない（失敗記録として保持）
```

```typescript
// 生成されるトランザクション
const result = await prisma.$transaction(async (tx) => {
  // 1. 注文作成
  const order = await tx.order.create({
    data: { userId, status: 'pending', total },
  });

  // 2. 注文アイテム一括作成
  await tx.orderItem.createMany({
    data: items.map(item => ({
      orderId: order.id,
      productId: item.productId,
      quantity: item.quantity,
      price: item.price,
    })),
  });

  // 3. 在庫を減らす（楽観的ロック: version確認）
  for (const item of items) {
    const updated = await tx.product.updateMany({
      where: { id: item.productId, stock: { gte: item.quantity } },
      data: { stock: { decrement: item.quantity } },
    });
    if (updated.count === 0) {
      throw new Error(`在庫不足: ${item.productId}`);
    }
  }

  // 4. ポイント使用
  await tx.user.update({
    where: { id: userId },
    data: { points: { decrement: pointsToUse } },
  });

  return order;
}, { timeout: 15000 });
```

---

## まとめ

Claude CodeでPrismaを正しく使う：

1. **CLAUDE.md** にシングルトン・N+1禁止・トランザクションルールを明記
2. **シングルトン** で開発時のClient過多を防ぐ
3. **include + select** でN+1を排除し、必要なフィールドだけ取得
4. **$transaction** で複数テーブル更新の整合性を保証

---

*PrismaのN+1・トランザクション設計の問題を自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
