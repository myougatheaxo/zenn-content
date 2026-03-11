---
title: "Claude Codeでマルチテナントを設計する：テナント分離とRow Level Security"
emoji: "🏢"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "security"]
published: true
---

## はじめに

マルチテナントSaaSで「テナントAのデータがテナントBから見える」というのは致命的なバグだ。Claude Codeにテナント分離を徹底した設計を生成させる。

---

## CLAUDE.mdにマルチテナントルールを書く

```markdown
## マルチテナント設計ルール

### テナント分離（必須）
- 全てのDBクエリにtenantIdフィルターを含める
- テナントIDはJWTから取得（リクエストパラメータ禁止）
- 全テーブルにtenantIdカラム（NOT NULL）必須

### ミドルウェア
- 認証ミドルウェアでtenantIdをreq.tenantに設定
- リポジトリ層でテナントフィルターを自動適用
- テナントコンテキストをAsyncLocalStorageで伝播

### Row Level Security (PostgreSQL)
- RLSポリシーでDB側でも分離を強制
- アプリ側ミスがあってもDB側で防御
```

---

## テナント分離の生成

```
マルチテナントの基盤を設計してください。

要件：
- JWTからtenantIdを取得してreq.tenantに設定
- Prismaの全クエリにtenantIdを自動付与（BaseRepositoryパターン）
- AsyncLocalStorageでテナントコンテキストを伝播
- PostgreSQL RLSポリシーでバックアップ分離

生成ファイル:
- src/middleware/tenant.ts
- src/repositories/baseRepository.ts
```

---

## 生成されるテナント分離基盤

```typescript
// src/middleware/tenant.ts
import { AsyncLocalStorage } from 'async_hooks';

export const tenantContext = new AsyncLocalStorage<{ tenantId: string }>();

export const tenantMiddleware = async (req: Request, res: Response, next: NextFunction) => {
  const tenantId = (req as AuthenticatedRequest).user?.tenantId;

  if (!tenantId) {
    return res.status(401).json({ error: 'Tenant context missing' });
  }

  // AsyncLocalStorageでテナントIDをスレッド（async context）に設定
  tenantContext.run({ tenantId }, () => {
    req.tenant = { id: tenantId };
    next();
  });
};

export function getCurrentTenantId(): string {
  const ctx = tenantContext.getStore();
  if (!ctx) throw new Error('Tenant context not set — missing middleware?');
  return ctx.tenantId;
}
```

```typescript
// src/repositories/baseRepository.ts
// 全リポジトリの基底クラス — テナントフィルターを自動付与

export abstract class BaseRepository<T> {
  constructor(protected readonly model: any) {}

  protected get tenantId(): string {
    return getCurrentTenantId();
  }

  async findById(id: string): Promise<T | null> {
    return this.model.findFirst({
      where: { id, tenantId: this.tenantId }, // tenantIdを必ず含める
    });
  }

  async findMany(where: Record<string, unknown> = {}): Promise<T[]> {
    return this.model.findMany({
      where: { ...where, tenantId: this.tenantId },
    });
  }

  async create(data: Record<string, unknown>): Promise<T> {
    return this.model.create({
      data: { ...data, tenantId: this.tenantId },
    });
  }

  async update(id: string, data: Record<string, unknown>): Promise<T> {
    // 更新前にテナント所有確認
    const existing = await this.findById(id);
    if (!existing) throw new NotFoundError(`${this.model.name} not found`);

    return this.model.update({
      where: { id },
      data,
    });
  }
}
```

```typescript
// src/repositories/userRepository.ts
export class UserRepository extends BaseRepository<User> {
  constructor() {
    super(prisma.user);
  }

  async findByEmail(email: string): Promise<User | null> {
    return prisma.user.findFirst({
      where: { email, tenantId: this.tenantId }, // 継承で自動付与
    });
  }
}
```

---

## PostgreSQL Row Level Security

```sql
-- RLSポリシー設定（バックアップ分離層）
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- アプリ側でセッション変数を設定
SET app.tenant_id = '550e8400-e29b-41d4-a716-446655440000';
```

```typescript
// Prisma + RLS: クエリ前にセッション変数を設定
prisma.$use(async (params, next) => {
  const tenantId = getCurrentTenantId();

  // Prismaの拡張でトランザクション内にRLS変数を設定
  await prisma.$executeRaw`SELECT set_config('app.tenant_id', ${tenantId}, true)`;

  return next(params);
});
```

---

## まとめ

Claude Codeでマルチテナントを設計する：

1. **CLAUDE.md** にtenantId必須・JWT取得・RLSを明記
2. **AsyncLocalStorage** でテナントコンテキストをスレッドに伝播
3. **BaseRepository** で全クエリに自動テナントフィルター付与
4. **PostgreSQL RLS** でアプリ側ミスをDB側でバックアップ防御

---

*テナント分離の脆弱性（テナント越え参照、JWTバイパス）を検出するスキルは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
