---
title: "Claude Codeで監査ログを設計する：誰が何をいつ変更したかを記録する"
emoji: "📔"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "security"]
published: true
---

## はじめに

「誰がこのレコードを変更したか」「いつ削除されたか」——監査ログがなければ追跡できない。金融・医療・SaaSでは必須のコンプライアンス要件でもある。Claude Codeに監査ログ設計を生成させる。

---

## CLAUDE.mdに監査ログルールを書く

```markdown
## 監査ログ設計ルール

### 記録対象（必須）
- 全てのCUD操作（Create/Update/Delete）を記録
- 管理者操作は特に詳細に（実行ユーザー、変更前後の値）
- 認証イベント: ログイン成功/失敗、パスワード変更、権限変更

### データ構造
- actor: 誰が（userId, email, role）
- action: 何を（created/updated/deleted）
- resource: 何に対して（entityType + entityId）
- changes: 何を変更（before/after）
- metadata: IPアドレス、UserAgent、リクエストID

### 保持期間
- 監査ログは変更禁止・削除禁止（immutable）
- 保持期間: 7年（金融）/ 5年（医療）/ 3年（一般SaaS）
- 古いログはS3/Glacierにアーカイブ
```

---

## 監査ログの生成

```
監査ログシステムを設計してください。

要件：
- Prisma middlewareで全CUD操作を自動キャプチャ
- 変更前後のデータをJSON記録
- ActorはAsyncLocalStorageからコンテキストで取得
- 非同期書き込み（本体処理をブロックしない）
- 検索API: actor/resource/dateRange でフィルタ

生成ファイル:
- src/audit/auditLogger.ts
- src/audit/auditMiddleware.ts
```

---

## 生成される監査ログ

```typescript
// src/audit/auditLogger.ts
import { AsyncLocalStorage } from 'async_hooks';

export interface AuditActor {
  userId: string;
  email: string;
  role: string;
  ipAddress: string;
  userAgent: string;
  requestId: string;
}

const actorContext = new AsyncLocalStorage<AuditActor>();

export function setAuditActor(actor: AuditActor, fn: () => void): void {
  actorContext.run(actor, fn);
}

export function getAuditActor(): AuditActor | undefined {
  return actorContext.getStore();
}

export async function logAuditEvent(params: {
  action: 'created' | 'updated' | 'deleted';
  entityType: string;
  entityId: string;
  before?: Record<string, unknown>;
  after?: Record<string, unknown>;
}): Promise<void> {
  const actor = getAuditActor();

  // 非同期書き込み（本体処理をブロックしない）
  setImmediate(async () => {
    try {
      await prisma.auditLog.create({
        data: {
          actorId: actor?.userId,
          actorEmail: actor?.email,
          actorRole: actor?.role,
          ipAddress: actor?.ipAddress,
          userAgent: actor?.userAgent,
          requestId: actor?.requestId,
          action: params.action,
          entityType: params.entityType,
          entityId: params.entityId,
          before: params.before ? JSON.stringify(params.before) : undefined,
          after: params.after ? JSON.stringify(params.after) : undefined,
        },
      });
    } catch (err) {
      // 監査ログ失敗は本体処理に影響させない（ただしアラート）
      logger.error({ err }, 'Failed to write audit log');
      await notifySlack(`⚠️ Audit log write failed: ${err}`);
    }
  });
}
```

```typescript
// src/audit/auditMiddleware.ts
// Prisma middlewareで全CUD操作を自動キャプチャ
prisma.$use(async (params, next) => {
  const auditable = ['create', 'update', 'updateMany', 'delete', 'deleteMany'];

  if (!auditable.includes(params.action)) {
    return next(params);
  }

  // 更新・削除前の値を取得
  let before: Record<string, unknown> | undefined;
  if (params.action === 'update' || params.action === 'delete') {
    const entity = await (prisma as any)[params.model!.toLowerCase()].findUnique({
      where: params.args.where,
    });
    before = entity ?? undefined;
  }

  const result = await next(params);

  const actionMap: Record<string, 'created' | 'updated' | 'deleted'> = {
    create: 'created',
    update: 'updated',
    updateMany: 'updated',
    delete: 'deleted',
    deleteMany: 'deleted',
  };

  await logAuditEvent({
    action: actionMap[params.action],
    entityType: params.model ?? 'unknown',
    entityId: result?.id ?? params.args.where?.id ?? 'bulk',
    before,
    after: params.action.startsWith('delete') ? undefined : result,
  });

  return result;
});
```

---

## Prismaスキーマ（監査ログテーブル）

```prisma
model AuditLog {
  id          String   @id @default(cuid())
  createdAt   DateTime @default(now())

  // Actor情報
  actorId     String?
  actorEmail  String?
  actorRole   String?
  ipAddress   String?
  userAgent   String?
  requestId   String?

  // アクション
  action      String   // 'created' | 'updated' | 'deleted'
  entityType  String   // 'User' | 'Order' | etc.
  entityId    String

  // 変更内容
  before      Json?
  after       Json?

  @@index([entityType, entityId])
  @@index([actorId, createdAt])
  @@index([createdAt])
}
```

---

## まとめ

Claude Codeで監査ログを設計する：

1. **CLAUDE.md** に全CUD記録必須・immutable・保持期間を明記
2. **Prisma middleware** で全操作を自動キャプチャ（漏れなし）
3. **AsyncLocalStorage** でリクエストコンテキスト（Actor）を自動伝播
4. **setImmediate** で非同期書き込み（本体処理をブロックしない）

---

*監査ログの設計レビュー（記録漏れ、機密情報の記録）は **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
