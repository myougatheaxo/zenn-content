---
title: "Claude CodeでRBAC（ロールベースアクセス制御）を設計する：権限管理・ポリシー・監査"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "security"]
published: true
---

## はじめに

「管理者は全て閲覧できるが、一般ユーザーは自分のデータだけ」——権限管理をad hocに実装すると漏れが生じる。RBACで権限を体系的に管理する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにRBAC設計ルールを書く

```markdown
## RBAC設計ルール

### ロール設計
- 最小権限の原則：必要最低限の権限のみ付与
- ロール階層：admin > manager > member > guest
- テナント横断ロール禁止（テナントごとにロール管理）

### 権限チェック
- コントローラー層で必ずロールチェック（Service層は行わない）
- リソースオーナーチェックは別途実装（ロールとは別）
- 権限不足は403（認証エラー401と区別）

### 監査
- 権限変更は必ず監査ログを記録
- 失敗したアクセス試行もログに残す
- 本番環境では権限変更に2人の承認を必要とする
```

---

## RBACシステムの生成

```
RBACシステムを設計してください。

エンティティ: User, Role, Permission, Resource
要件：
- ロール定義（admin/manager/member/guest）
- ミドルウェアで権限チェック
- リソースオーナーチェック
- Caslライブラリを使った宣言的権限定義

生成ファイル: src/authorization/
```

---

## 生成されるRBAC実装

```typescript
// src/authorization/abilities.ts
import { AbilityBuilder, createMongoAbility, MongoAbility } from '@casl/ability';

type Action = 'create' | 'read' | 'update' | 'delete' | 'manage';
type Subject = 'Order' | 'User' | 'Product' | 'Report' | 'all';

export type AppAbility = MongoAbility<[Action, Subject]>;

interface AuthUser {
  id: string;
  tenantId: string;
  role: 'admin' | 'manager' | 'member' | 'guest';
}

// ロールに応じた権限を定義
export function defineAbilitiesFor(user: AuthUser): AppAbility {
  const { can, cannot, build } = new AbilityBuilder<AppAbility>(createMongoAbility);

  switch (user.role) {
    case 'admin':
      // 管理者: 全リソースに全操作
      can('manage', 'all');
      break;

    case 'manager':
      // マネージャー: 注文の全操作、ユーザーの閲覧、商品の管理
      can('manage', 'Order');
      can('read', 'User');
      can('manage', 'Product');
      can('read', 'Report');
      // マネージャーでもユーザー削除は禁止
      cannot('delete', 'User');
      break;

    case 'member':
      // メンバー: 自分のリソースのみ
      can('read', 'Order', { userId: user.id });      // 自分の注文のみ閲覧
      can('create', 'Order');
      can('update', 'Order', { userId: user.id, status: 'pending' }); // 自分のpending注文のみ
      can('read', 'Product');
      break;

    case 'guest':
      // ゲスト: 商品閲覧のみ
      can('read', 'Product');
      break;
  }

  return build({
    detectSubjectType: (subject) => subject.__typename ?? subject.constructor.name,
  });
}
```

```typescript
// src/authorization/middleware.ts
import { subject } from '@casl/ability';

// ロール要件ミドルウェア（単純なロールチェック）
export function requireRole(...roles: AuthUser['role'][]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    if (!roles.includes(req.user.role)) {
      // アクセス失敗を監査ログに記録
      logger.warn({
        userId: req.user.id,
        requiredRoles: roles,
        userRole: req.user.role,
        path: req.path,
        method: req.method,
      }, 'Access denied: insufficient role');

      return res.status(403).json({
        error: 'Insufficient permissions',
        required: roles,
      });
    }

    next();
  };
}

// CASL権限ミドルウェア（リソースベースのきめ細かい制御）
export function authorize(action: Action, subjectType: Subject) {
  return async (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const ability = defineAbilitiesFor(req.user);

    // リソースインスタンスがある場合はオーナーチェックも含める
    const resourceId = req.params.id;
    if (resourceId) {
      const resource = await loadResource(subjectType, resourceId);
      if (!resource) {
        return res.status(404).json({ error: 'Resource not found' });
      }

      // subjectでリソースインスタンスを渡す（オーナーチェックが機能する）
      if (ability.cannot(action, subject(subjectType, resource))) {
        logger.warn({
          userId: req.user.id,
          action,
          subject: subjectType,
          resourceId,
        }, 'Access denied: CASL check failed');

        return res.status(403).json({ error: 'Forbidden' });
      }

      req.resource = resource;
    } else {
      if (ability.cannot(action, subjectType)) {
        return res.status(403).json({ error: 'Forbidden' });
      }
    }

    next();
  };
}
```

---

## ルートへの適用

```typescript
// src/routes/orders.ts

// 注文一覧: ロールチェック
router.get(
  '/orders',
  authenticate,
  requireRole('admin', 'manager'), // 管理者とマネージャーのみ全件閲覧
  async (req, res) => {
    const orders = await prisma.order.findMany();
    res.json(orders);
  }
);

// 注文詳細: リソースベース権限
router.get(
  '/orders/:id',
  authenticate,
  authorize('read', 'Order'),
  async (req, res) => {
    // req.resourceにはロードされた注文が入っている
    res.json(req.resource);
  }
);

// 注文更新: オーナーまたは管理者
router.patch(
  '/orders/:id',
  authenticate,
  authorize('update', 'Order'),
  async (req, res) => {
    const order = await prisma.order.update({
      where: { id: req.params.id },
      data: req.body,
    });
    res.json(order);
  }
);

// ユーザー削除: 管理者のみ
router.delete(
  '/users/:id',
  authenticate,
  authorize('delete', 'User'),
  async (req, res) => {
    await prisma.user.delete({ where: { id: req.params.id } });
    res.status(204).send();
  }
);
```

---

## DB schema（PostgreSQL）

```sql
-- ロール定義
CREATE TYPE user_role AS ENUM ('admin', 'manager', 'member', 'guest');

-- テナントごとにロールを管理
ALTER TABLE users ADD COLUMN role user_role NOT NULL DEFAULT 'member';
ALTER TABLE users ADD COLUMN tenant_id UUID NOT NULL REFERENCES tenants(id);

-- ロール変更の監査ログ
CREATE TABLE role_change_audit (
  id SERIAL PRIMARY KEY,
  target_user_id UUID NOT NULL REFERENCES users(id),
  changed_by_user_id UUID NOT NULL REFERENCES users(id),
  from_role user_role NOT NULL,
  to_role user_role NOT NULL,
  reason TEXT,
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## まとめ

Claude CodeでRBACを設計する：

1. **CLAUDE.md** に最小権限・テナント横断ロール禁止・権限変更の監査を明記
2. **Caslライブラリ** でロール定義を宣言的に記述（コードとして管理）
3. **subject()** でリソースインスタンスを渡してオーナーチェックも実装
4. **アクセス失敗を構造化ログ** で記録（監査トレイルとして活用）

---

*RBAC設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
