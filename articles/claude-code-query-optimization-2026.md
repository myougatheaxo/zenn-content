---
title: "Claude Codeでクエリ最適化を設計する：N+1問題・スロークエリ・インデックス戦略"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "prisma"]
published: true
---

## はじめに

「なぜか本番が遅い」——N+1問題、インデックス未設定、巨大なSELECT *。Claude Codeにクエリ最適化設計を生成させる。

---

## CLAUDE.mdにクエリ最適化ルールを書く

```markdown
## クエリ最適化ルール

### N+1問題
- リスト取得でループ内DBクエリ禁止
- Prismaではinclude/selectで関連データを一括取得
- DataLoader（GraphQL）またはfindManyでまとめて取得

### SELECT最適化
- SELECT * 禁止（必要なカラムのみ select で指定）
- 大量データのリスト: limit/cursor必須

### インデックス
- WHERE条件のカラムにindex
- 複合条件はcomposite index（順序重要）
- ユニーク制約は@@unique（インデックスも自動作成）

### スロークエリ
- 実行計画(EXPLAIN ANALYZE)で確認
- 500ms超のクエリはSlack通知
- Prisma: queryRawでEXPLAIN ANALYZEを取得
```

---

## N+1問題の修正

```
以下のN+1クエリを最適化してください。

問題のあるコード:
- ユーザーリストを取得（findMany）
- 各ユーザーの注文数をループでcount → N+1

修正方針:
- Prismaの _count で一括取得
- または生SQLでGROUP BY COUNT
```

```typescript
// ❌ N+1問題: ユーザー数分のクエリが実行される
const users = await prisma.user.findMany();
for (const user of users) {
  user.orderCount = await prisma.order.count({ where: { userId: user.id } }); // N回実行
}

// ✅ 修正: _count で一括取得（1クエリ）
const users = await prisma.user.findMany({
  include: {
    _count: {
      select: { orders: true },
    },
  },
});
// users[0]._count.orders でアクセス可能
```

```typescript
// ❌ N+1問題: 投稿リストで各著者を個別取得
const posts = await prisma.post.findMany();
for (const post of posts) {
  post.author = await prisma.user.findUnique({ where: { id: post.authorId } });
}

// ✅ 修正: includeで一括取得（2クエリ: posts + IN句でusers）
const posts = await prisma.post.findMany({
  include: {
    author: {
      select: { id: true, name: true, avatarUrl: true }, // 必要なフィールドのみ
    },
  },
});
```

---

## スロークエリの検出

```typescript
// Prismaのミドルウェアでスロークエリを検出
prisma.$use(async (params, next) => {
  const start = Date.now();
  const result = await next(params);
  const duration = Date.now() - start;

  if (duration > 500) {
    logger.warn({
      model: params.model,
      action: params.action,
      durationMs: duration,
    }, 'Slow query detected');

    if (duration > 2000) {
      await notifySlack(`🐌 Slow query: ${params.model}.${params.action} took ${duration}ms`);
    }
  }

  return result;
});
```

---

## EXPLAIN ANALYZE で実行計画を確認

```typescript
// Prismaで生SQLのEXPLAIN ANALYZEを実行
async function explainQuery(sql: string, params: unknown[]): Promise<void> {
  const explain = await prisma.$queryRawUnsafe(
    `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
    ...params
  );

  const plan = (explain as Array<{ 'QUERY PLAN': unknown[] }>)[0]['QUERY PLAN'];
  console.log(JSON.stringify(plan, null, 2));
}

// 使用例: スロークエリの実行計画を確認
await explainQuery(
  'SELECT * FROM orders WHERE user_id = $1 AND status = $2',
  ['user_123', 'pending']
);
```

---

## Prismaスキーマのインデックス設計

```prisma
model Order {
  id        String   @id @default(cuid())
  userId    String
  status    String   // 'pending' | 'completed' | 'cancelled'
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id])

  // WHERE userId = ? AND status = ? でよく検索する場合
  @@index([userId, status])

  // WHERE createdAt > ? ORDER BY createdAt DESC でよく使う場合
  @@index([createdAt(sort: Desc)])

  // 複合ユニーク制約（インデックス自動作成）
  @@unique([userId, externalOrderId])
}
```

---

## まとめ

Claude Codeでクエリ最適化を設計する：

1. **CLAUDE.md** にN+1禁止・SELECT*禁止・インデックスルールを明記
2. **Prisma include/_count** でN+1を排除
3. **スロークエリ監視** Prisma middlewareで500ms超を検出→Slack通知
4. **EXPLAIN ANALYZE** で実行計画を確認してインデックス効果を検証

---

*N+1問題やインデックス未設定を検出するスキルは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
