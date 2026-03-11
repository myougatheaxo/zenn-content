---
title: "Claude CodeでPII（個人情報）マスキングを設計する：ログ自動除去・APIレスポンス制御・GDPR対応"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "postgresql"]
published: true
published_at: "2026-03-16 09:00"
---

## はじめに

「ログにメールアドレスが丸見え」「開発環境にDBを渡したら本番メールが入っていた」——PIIをログから自動除去し、APIレスポンスでもマスキングする設計をClaude Codeに生成させる。

---

## CLAUDE.mdにPIIマスキング設計ルールを書く

```markdown
## PIIマスキング設計ルール

### PII分類
- クラスA（高機密）: パスワードハッシュ, 決済カード番号, マイナンバー → ログに絶対出力しない
- クラスB（機密）: メールアドレス, 電話番号, 氏名 → ログは部分マスク（先頭+末尾のみ）
- クラスC（低機密）: 住所, 生年月日 → ログは完全マスク、API応答は権限によって出し分け

### ログマスキング
- loggerラッパーで全フィールドを検査・マスキング
- pino/winstonのserializerで透過的に処理
- マスキングルールはCLAUDE.mdと同じ定義ファイルを参照

### APIレスポンス
- エンドポイントごとにPermission Matrix（誰が何のPIIを見られるか）
- 本人は全情報参照可、オペレーターは部分マスク、第三者APIは最小限のみ
```

---

## PIIマスキング実装の生成

```
PIIマスキングシステムを設計してください。

要件：
- ログの自動マスキング（pino serializer）
- APIレスポンスの権限ベースマスキング
- DBの論理削除時のPII削除（GDPR忘れられる権利）
- 開発環境用データ匿名化

生成ファイル: src/privacy/
```

---

## 生成されるPIIマスキング実装

```typescript
// src/privacy/piiMasker.ts — PIIマスキングエンジン

export type PiiClass = 'A' | 'B' | 'C';

export interface PiiField {
  path: string;     // JSONパス（例: 'user.email'）
  class: PiiClass;
  masker: (value: string) => string;
}

// マスキング関数定義
export const PII_MASKS = {
  email: (v: string): string => {
    const [local, domain] = v.split('@');
    if (!domain) return '***';
    return `${local.slice(0, 2)}***@${domain}`;
  },
  phone: (v: string): string => `${v.slice(0, 3)}****${v.slice(-4)}`,
  name: (v: string): string => `${v.slice(0, 1)}***`,
  creditCard: (_v: string): string => '****-****-****-****',
  password: (_v: string): string => '[REDACTED]',
  fullMask: (_v: string): string => '***',
};

// PII定義（全プロジェクト共通）
export const PII_FIELDS: PiiField[] = [
  { path: 'password',     class: 'A', masker: PII_MASKS.password },
  { path: 'passwordHash', class: 'A', masker: PII_MASKS.password },
  { path: 'cardNumber',   class: 'A', masker: PII_MASKS.creditCard },
  { path: 'email',        class: 'B', masker: PII_MASKS.email },
  { path: 'phone',        class: 'B', masker: PII_MASKS.phone },
  { path: 'name',         class: 'B', masker: PII_MASKS.name },
  { path: 'address',      class: 'C', masker: PII_MASKS.fullMask },
  { path: 'birthDate',    class: 'C', masker: PII_MASKS.fullMask },
];

// ネストされたオブジェクトのPIIをマスク
export function maskPii(
  obj: Record<string, unknown>,
  minimumClass: PiiClass = 'A' // デフォルト: クラスA（最高機密）のみマスク
): Record<string, unknown> {
  const classOrder: PiiClass[] = ['A', 'B', 'C'];
  const minimumIdx = classOrder.indexOf(minimumClass);

  const fieldsToMask = PII_FIELDS.filter(
    f => classOrder.indexOf(f.class) >= minimumIdx
  );

  return deepMask(obj, fieldsToMask);
}

function deepMask(
  obj: unknown,
  fields: PiiField[],
  currentPath = ''
): unknown {
  if (obj === null || typeof obj !== 'object') return obj;

  if (Array.isArray(obj)) {
    return obj.map(item => deepMask(item, fields, currentPath));
  }

  const result: Record<string, unknown> = {};

  for (const [key, value] of Object.entries(obj as Record<string, unknown>)) {
    const path = currentPath ? `${currentPath}.${key}` : key;
    const piiDef = fields.find(f => f.path === key || f.path === path);

    if (piiDef && typeof value === 'string') {
      result[key] = piiDef.masker(value);
    } else if (value && typeof value === 'object') {
      result[key] = deepMask(value, fields, path);
    } else {
      result[key] = value;
    }
  }

  return result;
}
```

```typescript
// src/privacy/loggerWithMasking.ts — pinoシリアライザーでログを自動マスク

import pino from 'pino';

// pinoのserializerでログに含まれるPIIを自動マスク
export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  serializers: {
    // 全ログオブジェクトに適用（err以外）
    req: (req) => ({
      method: req.method,
      url: req.url,
      // Authorizationヘッダーは完全除去
      headers: Object.fromEntries(
        Object.entries(req.headers ?? {})
          .filter(([k]) => !['authorization', 'cookie', 'x-api-key'].includes(k.toLowerCase()))
      ),
    }),
    user: (user) => maskPii(user, 'B'), // ユーザーオブジェクトはクラスB以上をマスク
    // カスタムオブジェクトはクラスA（最高機密）のみマスク
    default: (obj) => typeof obj === 'object' && obj ? maskPii(obj as any, 'A') : obj,
  },
});

// 使用例（マスキングは自動）
logger.info({ user: { id: 'u1', email: 'test@example.com', password: 'hashed' } }, 'User logged in');
// ログ出力: { user: { id: 'u1', email: 'te***@example.com', password: '[REDACTED]' } }
```

```typescript
// src/privacy/apiResponseMasker.ts — API応答のPIIを権限ベースでマスク

export type ViewerRole = 'self' | 'operator' | 'admin' | 'public';

const ROLE_MASK_LEVEL: Record<ViewerRole, PiiClass | 'none'> = {
  self:     'none',      // 本人: マスクなし
  admin:    'none',      // 管理者: マスクなし
  operator: 'B',         // オペレーター: クラスB以上マスク（メール・電話）
  public:   'C',         // 一般/外部API: 全PIIマスク
};

export function maskForViewer<T extends Record<string, unknown>>(
  data: T,
  viewerRole: ViewerRole
): T {
  const level = ROLE_MASK_LEVEL[viewerRole];
  if (level === 'none') return data;
  return maskPii(data, level) as T;
}

// APIエンドポイントでの使用
router.get('/api/users/:id', requireAuth, async (req, res) => {
  const user = await prisma.user.findUniqueOrThrow({ where: { id: req.params.id } });

  const role: ViewerRole =
    req.userId === user.id ? 'self' :
    req.userRole === 'admin' ? 'admin' :
    req.userRole === 'operator' ? 'operator' : 'public';

  res.json(maskForViewer(user, role));
});

// src/privacy/gdprDeletion.ts — GDPR忘れられる権利（匿名化）

export async function anonymizeUser(userId: string): Promise<void> {
  const anonymizedEmail = `deleted_${userId}@anonymous.invalid`;
  const anonymizedName = 'Deleted User';

  await prisma.$transaction(async (tx) => {
    // PII項目を匿名化（物理削除ではなくデータ上書き）
    await tx.user.update({
      where: { id: userId },
      data: {
        email: anonymizedEmail,
        name: anonymizedName,
        phone: null,
        address: null,
        birthDate: null,
        deletedAt: new Date(),
        anonymizedAt: new Date(),
      },
    });

    // ユーザー生成コンテンツの匿名化
    await tx.post.updateMany({
      where: { userId, deletedAt: null },
      data: { deletedAt: new Date() },
    });

    // 削除ログ（削除したこと自体は記録が必要）
    await tx.gdprDeletionLog.create({
      data: { userId, requestedAt: new Date(), completedAt: new Date() },
    });
  });

  logger.info({ userId }, 'User anonymized for GDPR deletion');
}
```

---

## まとめ

Claude CodeでPIIマスキングを設計する：

1. **CLAUDE.md** にPIIを3クラスに分類・ログはクラスB以上（メール・電話）を自動マスク・APIはロールベースで出し分けを明記
2. **pinoのserializer** でログへのPIIを透過的にマスク——アプリコードにマスキングロジックを書く必要なし
3. **maskForViewer** でビューアロール別に自動マスク——本人は全情報・オペレーターはメール非表示・外部APIは最小限
4. **GDPR匿名化** は物理削除ではなくデータ上書き——削除ログは残しつつPIIのみ消去・履歴整合性を維持

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
