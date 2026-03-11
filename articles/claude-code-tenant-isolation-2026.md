---
title: "Claude Codeでテナント分離を設計する：Row-Level Security・スキーマ分離・データリーク防止"
emoji: "🏢"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "security"]
published: true
published_at: "2026-03-13 10:00"
---

## はじめに

マルチテナントSaaSで最も怖いのはテナント間のデータリーク——PostgreSQL Row-Level SecurityとPrismaのグローバルミドルウェアで誤ったデータアクセスを物理的に防ぐ。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにテナント分離設計ルールを書く

```markdown
## テナント分離設計ルール

### 分離モデル
- 小規模テナント: Row-Level Security（PostgreSQL RLS）
- 大規模テナント: スキーマ分離（schema per tenant）
- エンタープライズ: DB分離（db per tenant）

### Row-Level Security（RLS）
- 全テーブルにtenant_idカラム必須
- PostgreSQL RLSポリシーで物理的に分離
- アプリケーション層での誤りをDBレベルで防ぐ

### テナントコンテキスト
- リクエストごとにtenant_idをAsyncLocalStorageに設定
- PrismaミドルウェアでWHERE句を自動追加
- tenant_idなしクエリはエラー（開発環境で即座に検知）
```

---

## テナント分離システムの生成

```
マルチテナントデータ分離を設計してください。

要件：
- PostgreSQL Row-Level Security
- PrismaのテナントコンテキストMiddleware
- テナントコンテキストの伝播
- クロステナントアクセス防止

生成ファイル: src/tenant/
```

---

## 生成されるテナント分離実装

```sql
-- migrations/001_add_rls.sql
-- PostgreSQL Row-Level Securityの設定

-- テナントIDのセットアップ
ALTER DATABASE myapp SET app.tenant_id = '';

-- ordersテーブルのRLSポリシー
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- SELECT/INSERT/UPDATE/DELETE全てにポリシーを適用
CREATE POLICY tenant_isolation_policy ON orders
  USING (tenant_id = current_setting('app.tenant_id')::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);

-- 管理者ロールはバイパス（マイグレーション用）
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
CREATE ROLE app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO app_user;

-- 他の全テーブルにも同様のポリシーを適用
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_policy ON products
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- usersテーブル（tenant_idで分離）
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_policy ON users
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

```typescript
// src/tenant/tenantContext.ts — テナントコンテキスト管理

import { AsyncLocalStorage } from 'async_hooks';

interface TenantContext {
  tenantId: string;
  tenantSlug: string;
  planId: string;
}

const tenantStorage = new AsyncLocalStorage<TenantContext>();

// 現在のテナントコンテキストを取得
export function getCurrentTenant(): TenantContext {
  const ctx = tenantStorage.getStore();
  if (!ctx) throw new Error('No tenant context found. Did you call withTenantContext?');
  return ctx;
}

// テナントコンテキストを設定して実行
export function withTenantContext<T>(
  context: TenantContext,
  fn: () => T
): T {
  return tenantStorage.run(context, fn);
}

// Expressミドルウェア: リクエストからテナントを解決
export async function tenantMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  // サブドメインかヘッダーからテナントを特定
  const tenantSlug = extractTenantSlug(req);

  if (!tenantSlug) {
    return res.status(400).json({ error: 'Tenant not specified' });
  }

  // テナント情報をキャッシュから取得
  const cacheKey = `tenant:${tenantSlug}`;
  const cached = await redis.get(cacheKey);
  const tenant = cached ? JSON.parse(cached) : await prisma.tenant.findUnique({ where: { slug: tenantSlug } });

  if (!tenant) {
    return res.status(404).json({ error: 'Tenant not found' });
  }

  if (!cached) {
    await redis.set(cacheKey, JSON.stringify(tenant), { EX: 300 }); // 5分キャッシュ
  }

  // テナントコンテキストを設定してリクエストを処理
  withTenantContext({ tenantId: tenant.id, tenantSlug: tenant.slug, planId: tenant.planId }, () => {
    next();
  });
}

function extractTenantSlug(req: Request): string | null {
  // サブドメインから取得: tenant1.myapp.com
  const host = req.hostname;
  const subdomain = host.split('.')[0];
  if (subdomain && subdomain !== 'www' && subdomain !== 'api') return subdomain;

  // ヘッダーから取得: X-Tenant-Slug
  return req.headers['x-tenant-slug'] as string | null;
}
```

```typescript
// src/tenant/prismaMiddleware.ts — Prismaで自動テナントフィルタリング

// PrismaのMiddlewareでSELECT/INSERT/UPDATE/DELETE全てにtenant_idを自動付与
export function setupTenantMiddleware(prisma: PrismaClient): void {
  // 1. PostgreSQL RLSのtenant_id設定
  prisma.$use(async (params, next) => {
    const ctx = tenantStorage.getStore();
    if (!ctx) return next(params); // システム処理はコンテキストなし

    // PostgreSQL session変数にtenant_idをセット（RLSが参照）
    await prisma.$executeRaw`SELECT set_config('app.tenant_id', ${ctx.tenantId}, true)`;

    return next(params);
  });

  // 2. Prismaクエリにtenant_idを自動追加（アプリケーション層の二重防衛）
  const TENANT_TABLES = ['Order', 'Product', 'User', 'Invoice'];

  prisma.$use(async (params, next) => {
    const ctx = tenantStorage.getStore();
    if (!ctx || !TENANT_TABLES.includes(params.model ?? '')) return next(params);

    // findの場合はwhereにtenant_idを追加
    if (['findFirst', 'findMany', 'findUnique', 'count'].includes(params.action)) {
      params.args.where = { ...params.args.where, tenantId: ctx.tenantId };
    }

    // create時にtenant_idを自動付与
    if (params.action === 'create') {
      params.args.data = { ...params.args.data, tenantId: ctx.tenantId };
    }

    // update/deleteでもtenant_idフィルターを追加
    if (['update', 'updateMany', 'delete', 'deleteMany'].includes(params.action)) {
      params.args.where = { ...params.args.where, tenantId: ctx.tenantId };
    }

    return next(params);
  });
}

// テナント分離違反の検知（開発環境）
prisma.$use(async (params, next) => {
  if (process.env.NODE_ENV === 'development') {
    const ctx = tenantStorage.getStore();
    const TENANT_TABLES = ['Order', 'Product', 'User'];

    if (TENANT_TABLES.includes(params.model ?? '') && !ctx) {
      // スタックトレース付きでエラー
      throw new Error(
        `Tenant context required for ${params.model}.${params.action}. ` +
        `Did you forget to call withTenantContext()?`
      );
    }
  }
  return next(params);
});
```

---

## テナント別プラン制限

```typescript
// src/tenant/planLimits.ts — プランに応じた機能制限

const PLAN_LIMITS: Record<string, PlanLimits> = {
  free:       { maxUsers: 5,  maxStorage: 1_000, apiCallsPerMonth: 1_000 },
  pro:        { maxUsers: 50, maxStorage: 100_000, apiCallsPerMonth: 100_000 },
  enterprise: { maxUsers: Infinity, maxStorage: Infinity, apiCallsPerMonth: Infinity },
};

export async function checkPlanLimit(
  tenantId: string,
  resource: keyof PlanLimits
): Promise<void> {
  const tenant = await getTenantWithUsage(tenantId);
  const limits = PLAN_LIMITS[tenant.plan];

  if (tenant.usage[resource] >= limits[resource]) {
    throw new PlanLimitError(
      `${resource} limit reached for ${tenant.plan} plan. Upgrade to increase limits.`,
      { resource, current: tenant.usage[resource], limit: limits[resource] }
    );
  }
}
```

---

## まとめ

Claude CodeでテナントデータIsolationを設計する：

1. **CLAUDE.md** にRLS必須・全テーブルtenant_id・PrismaMiddlewareで自動フィルタを明記
2. **PostgreSQL RLSポリシー** でDBレベルのデータ分離（アプリのバグによるリーク防止）
3. **PrismaMiddleware** で全クエリに自動的にtenant_idフィルターを付与（二重防衛）
4. **開発環境でテナントコンテキストなしエラー** を即座に検知してバグを早期発見

---

*テナント分離設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
