---
title: "Claude CodeでSoft Deleteパターンを設計する：論理削除・一括復元・削除後TTL"
emoji: "🗑️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "prisma", "postgresql"]
published: true
published_at: "2026-03-15 13:00"
---

## はじめに

「間違えて消してしまった」を30日以内なら復元できる——`deleted_at`タイムスタンプで論理削除を実装し、誤削除からの復元・完全削除のスケジューリングをClaude Codeに設計させる。

---

## CLAUDE.mdにSoft Delete設計ルールを書く

```markdown
## Soft Delete設計ルール

### 実装方式
- deleted_at: TIMESTAMPTZ NOT NULL DEFAULT NULL
- Prismaミドルウェアで全クエリに WHERE deleted_at IS NULL を自動付与
- 物理削除は絶対禁止（ユーザー操作からは）

### 復元ポリシー
- 復元期間: 30日
- 30日後: 完全削除（物理削除・S3ファイルも削除）
- 復元は元のデータを完全に戻す（deleted_atをNULLに）

### カスケード削除
- 親を削除すると子も論理削除（同時刻のdeleted_atで識別）
- 復元時は子も同時に復元（同じdeleted_atのレコードを対象）
- 子のみの独立削除はdeleted_atが異なる
```

---

## Soft Delete実装の生成

```
Soft Delete（論理削除）システムを設計してください。

要件：
- Prismaミドルウェアで自動フィルタ
- カスケード論理削除
- 30日後の完全削除スケジュール
- 復元機能

生成ファイル: src/db/softDelete/
```

---

## 生成されるSoft Delete実装

```typescript
// src/db/softDelete/prismaMiddleware.ts — 全クエリに自動フィルタ

import { Prisma } from '@prisma/client';

// soft-deleteフィルタが必要なモデル一覧
const SOFT_DELETE_MODELS = new Set([
  'User', 'Post', 'Order', 'Product', 'File',
]);

export function softDeleteMiddleware(): Prisma.Middleware {
  return async (params, next) => {
    // soft-deleteモデルのみ処理
    if (!params.model || !SOFT_DELETE_MODELS.has(params.model)) {
      return next(params);
    }

    // findMany / findFirst / findUnique への自動フィルタ付与
    if (['findMany', 'findFirst', 'findUnique', 'findFirstOrThrow', 'findUniqueOrThrow'].includes(params.action)) {
      // withDeletedフラグがある場合はフィルタをスキップ
      if (params.args?.where?.withDeleted) {
        delete params.args.where.withDeleted;
        return next(params);
      }

      params.args = params.args ?? {};
      params.args.where = params.args.where ?? {};

      if (params.action === 'findUnique' || params.action === 'findUniqueOrThrow') {
        // findUniqueはwhereを直接変更できないためfindFirstに変換
        params.action = params.action === 'findUniqueOrThrow' ? 'findFirstOrThrow' : 'findFirst';
        params.args.where.deletedAt = null;
      } else {
        params.args.where.deletedAt = null;
      }
    }

    // count にも自動フィルタ
    if (params.action === 'count') {
      params.args = params.args ?? {};
      params.args.where = { ...(params.args.where ?? {}), deletedAt: null };
    }

    // delete → update（deletedAtを設定）
    if (params.action === 'delete') {
      params.action = 'update';
      params.args.data = { deletedAt: new Date() };
    }

    // deleteMany → updateMany
    if (params.action === 'deleteMany') {
      params.action = 'updateMany';
      params.args.data = { deletedAt: new Date() };
    }

    return next(params);
  };
}
```

```typescript
// src/db/softDelete/cascadeDelete.ts — カスケード論理削除

export async function softDeleteWithCascade(
  model: 'Post',
  id: string,
  userId: string
): Promise<void> {
  const deletedAt = new Date(); // 同一タイムスタンプで子も削除

  await prisma.$transaction(async (tx) => {
    // 所有権確認
    const record = await tx.post.findFirst({ where: { id, userId, deletedAt: null } });
    if (!record) throw new NotFoundError(`Post ${id} not found`);

    // 親を論理削除
    await tx.post.update({
      where: { id },
      data: { deletedAt },
    });

    // 子コメントも同じタイムスタンプで論理削除（カスケード）
    await tx.comment.updateMany({
      where: { postId: id, deletedAt: null },
      data: { deletedAt },
    });

    // 添付ファイルも削除
    await tx.file.updateMany({
      where: { postId: id, deletedAt: null },
      data: { deletedAt },
    });

    // 削除ログ記録
    await tx.deletionLog.create({
      data: { modelName: 'Post', recordId: id, deletedBy: userId, deletedAt },
    });
  });

  logger.info({ postId: id, userId }, 'Post soft-deleted with cascade');
}

// カスケード復元（同一deletedAtのレコードを全て復元）
export async function restoreWithCascade(
  model: 'Post',
  id: string,
  userId: string
): Promise<void> {
  // 削除レコードを確認（withDeletedフラグでフィルタをバイパス）
  const record = await prisma.post.findFirst({
    where: { id, userId, withDeleted: true } as any,
  });

  if (!record?.deletedAt) throw new NotFoundError(`Deleted post ${id} not found`);

  // 30日以内かチェック
  const deletedDaysAgo = (Date.now() - record.deletedAt.getTime()) / (1000 * 60 * 60 * 24);
  if (deletedDaysAgo > 30) {
    throw new Error('Cannot restore: record deleted more than 30 days ago');
  }

  const deletedAt = record.deletedAt; // 同一タイムスタンプのものを復元対象に

  await prisma.$transaction(async (tx) => {
    // 親を復元
    await (tx as any).post.update({ where: { id }, data: { deletedAt: null } });

    // 同じdeleted_atの子コメントも復元（独立削除分は対象外）
    await (tx as any).comment.updateMany({
      where: { postId: id, deletedAt },
      data: { deletedAt: null },
    });

    await (tx as any).file.updateMany({
      where: { postId: id, deletedAt },
      data: { deletedAt: null },
    });
  });

  logger.info({ postId: id, userId }, 'Post restored with cascade');
}
```

```typescript
// src/db/softDelete/cleanupJob.ts — 30日後の完全削除スケジュール

// CronJob: 毎日午前2時に実行
export async function permanentDeleteExpired(): Promise<void> {
  const RETENTION_DAYS = 30;
  const cutoffDate = new Date(Date.now() - RETENTION_DAYS * 24 * 60 * 60 * 1000);

  logger.info({ cutoffDate }, 'Starting permanent delete of expired soft-deleted records');

  // 削除順: 子 → 親（FK制約の逆順）
  const fileResult = await prisma.file.deleteMany({
    where: { deletedAt: { lt: cutoffDate, not: null } },
  });

  // S3ファイルも削除
  const expiredFiles = await prisma.file.findMany({
    where: { deletedAt: { lt: cutoffDate, not: null } },
    select: { s3Key: true },
  });
  for (const file of expiredFiles) {
    await s3.deleteObject({ Bucket: process.env.S3_BUCKET!, Key: file.s3Key }).promise();
  }

  const commentResult = await prisma.comment.deleteMany({
    where: { deletedAt: { lt: cutoffDate, not: null } },
  });

  const postResult = await prisma.post.deleteMany({
    where: { deletedAt: { lt: cutoffDate, not: null } },
  });

  logger.info({
    posts: postResult.count,
    comments: commentResult.count,
    files: fileResult.count,
  }, 'Permanent delete completed');
}
```

---

## まとめ

Claude CodeでSoft Deleteパターンを設計する：

1. **CLAUDE.md** に全クエリへdeleted_at自動フィルタ・delete→updateに変換・30日後完全削除を明記
2. **Prismaミドルウェア** でdelete/deleteManyをupdateに変換——アプリ側から物理削除を不可能に
3. **カスケード削除** は親子で同一のdeletedAtタイムスタンプを使用——復元時に同タイムスタンプで一括特定
4. **完全削除CronJob** で30日経過レコードをFK逆順（子→親）で物理削除しS3ファイルも合わせて削除

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
