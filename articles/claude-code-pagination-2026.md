---
title: "Claude Codeでページネーションを設計する：カーソルベースと無限スクロール"
emoji: "📄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "prisma", "api"]
published: true
---

## はじめに

OFFSETページネーションは100万件のテーブルで遅くなる。カーソルベースページネーションに切り替えると大幅に改善できる。Claude Codeに安全な設計を生成させる。

---

## CLAUDE.mdにページネーションルールを書く

```markdown
## ページネーション設計ルール

### 方式
- 1万件超のデータ: カーソルベース必須（OFFSET禁止）
- 管理画面等の小規模: OFFSETページネーション可
- 無限スクロール: カーソルベース + hasNextPage

### カーソルベース
- カーソルはopaque（クライアントに内部構造を見せない）
- cursorはBase64エンコードしたIDまたは複合キー
- レスポンスに nextCursor と hasNextPage を含める

### パフォーマンス
- pageSize上限を設ける（最大100）
- orderBy必須（cursor基準のソートが必要）
- cursor対象カラムにindexを貼る
```

---

## カーソルページネーションの生成

```
カーソルベースのページネーションAPIを生成してください。

エンドポイント: GET /api/users
パラメータ: cursor?, limit(1-100, default 20)
レスポンス: { data: User[], nextCursor: string | null, hasNextPage: boolean }

要件：
- Prismaでカーソルページネーション
- cursorはBase64でエンコード
- orderBy: createdAt desc, id desc（タイ処理用）
```

---

## 生成されるカーソルページネーション

```typescript
// src/utils/pagination.ts
export function encodeCursor(id: string, createdAt: Date): string {
  return Buffer.from(JSON.stringify({ id, createdAt: createdAt.toISOString() })).toString('base64');
}

export function decodeCursor(cursor: string): { id: string; createdAt: Date } | null {
  try {
    const decoded = JSON.parse(Buffer.from(cursor, 'base64').toString('utf-8'));
    return { id: decoded.id, createdAt: new Date(decoded.createdAt) };
  } catch {
    return null;
  }
}

export interface PaginatedResult<T> {
  data: T[];
  nextCursor: string | null;
  hasNextPage: boolean;
}
```

```typescript
// src/services/userService.ts
export async function listUsers(
  limit: number,
  cursor?: string
): Promise<PaginatedResult<User>> {
  const take = Math.min(limit, 100); // 上限100
  const decoded = cursor ? decodeCursor(cursor) : null;

  const users = await prisma.user.findMany({
    take: take + 1, // 1件多く取得してhasNextPageを判定
    ...(decoded && {
      cursor: { id: decoded.id },
      skip: 1, // cursor自身をスキップ
    }),
    orderBy: [
      { createdAt: 'desc' },
      { id: 'desc' }, // 同一時刻のタイブレーカー
    ],
  });

  const hasNextPage = users.length > take;
  const data = hasNextPage ? users.slice(0, take) : users;
  const lastItem = data[data.length - 1];

  return {
    data,
    nextCursor: lastItem ? encodeCursor(lastItem.id, lastItem.createdAt) : null,
    hasNextPage,
  };
}
```

```typescript
// src/routes/users.ts
router.get('/users', authenticate, async (req, res) => {
  const limit = Math.max(1, Math.min(100, Number(req.query.limit) || 20));
  const cursor = req.query.cursor as string | undefined;

  const result = await listUsers(limit, cursor);
  res.json(result);
});
```

---

## フロントエンド（無限スクロール）

```typescript
// useInfiniteUsers.ts
import { useInfiniteQuery } from '@tanstack/react-query';

export function useInfiniteUsers() {
  return useInfiniteQuery({
    queryKey: ['users'],
    queryFn: ({ pageParam }) =>
      fetch(`/api/users?limit=20${pageParam ? `&cursor=${pageParam}` : ''}`).then(r => r.json()),
    getNextPageParam: (lastPage) => lastPage.hasNextPage ? lastPage.nextCursor : undefined,
    initialPageParam: undefined,
  });
}

// コンポーネント
function UserList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteUsers();
  const ref = useRef(null);

  useIntersectionObserver(ref, () => {
    if (hasNextPage && !isFetchingNextPage) fetchNextPage();
  });

  return (
    <>
      {data?.pages.flatMap(p => p.data).map(user => <UserCard key={user.id} user={user} />)}
      <div ref={ref} />
    </>
  );
}
```

---

## OFFSETページネーション（管理画面向け）

```typescript
// 管理画面等で「3ページ目」に直接ジャンプする必要がある場合のみ
router.get('/admin/users', async (req, res) => {
  const page = Math.max(1, Number(req.query.page) || 1);
  const limit = 20;

  const [total, users] = await prisma.$transaction([
    prisma.user.count(),
    prisma.user.findMany({
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { createdAt: 'desc' },
    }),
  ]);

  res.json({
    data: users,
    total,
    page,
    totalPages: Math.ceil(total / limit),
  });
});
```

---

## まとめ

Claude Codeでページネーションを設計する：

1. **CLAUDE.md** に1万件超はカーソルベース必須、OFFSET禁止を明記
2. **カーソルはBase64** でクライアントに内部構造を隠す
3. **take+1** で1件余分に取得してhasNextPageを判定
4. **React Query useInfiniteQuery** で無限スクロールを実装

---

*ページネーションの実装問題（OFFSETの大量OFFSET、型安全でないcursor）を検出するスキルは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
