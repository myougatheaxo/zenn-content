---
title: "Claude CodeでGDPR・個人情報保護を設計する：データ匿名化・削除権・同意管理"
emoji: "🔏"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "security"]
published: true
---

## はじめに

GDPR・個人情報保護法でユーザーは「自分のデータを削除して」と要求できる。ただし注文履歴などは会計上保持義務がある。データをマスクしながら業務継続する実装をClaude Codeに設計させる。

---

## CLAUDE.mdにデータプライバシー設計ルールを書く

```markdown
## GDPRデータプライバシー設計ルール

### データ分類
- カテゴリA（識別可能）: 氏名、メール、電話番号、住所 → 削除/マスク対象
- カテゴリB（業務必要）: 注文ID、取引金額、日時 → 保持（匿名化）
- カテゴリC（分析データ）: 行動ログ、アクセス履歴 → 削除対象

### 削除権（Right to Erasure）
- ユーザーデータは「削除」ではなく「匿名化」（業務継続のため）
- 匿名化後は元の個人とひも付け不可能にする
- 削除完了まで30日以内

### 同意管理
- 目的ごとに同意を分離（マーケティング・分析・必須）
- 同意の撤回はいつでも可能
- 同意履歴を保持（証跡）
```

---

## GDPRデータ管理システムの生成

```
GDPRに対応したユーザーデータ管理システムを設計してください。

要件：
- データ削除権（匿名化）
- データポータビリティ（エクスポート）
- 同意管理
- 監査ログ
- データ保持ポリシー

生成ファイル: src/privacy/
```

---

## 生成されるGDPRデータ管理実装

```typescript
// src/privacy/dataAnonymizer.ts

interface AnonymizationResult {
  affectedTables: string[];
  recordsAnonymized: number;
  completedAt: Date;
}

// ユーザーデータの匿名化（削除ではなく置換）
export async function anonymizeUser(userId: string, requestedBy: string): Promise<AnonymizationResult> {
  const anonymousId = `deleted_${crypto.randomBytes(8).toString('hex')}`;
  const anonymousEmail = `${anonymousId}@anonymized.local`;
  const affectedTables: string[] = [];
  let recordsAnonymized = 0;

  await prisma.$transaction(async (tx) => {
    // usersテーブル: 識別可能情報をマスク
    const userUpdate = await tx.user.update({
      where: { id: userId },
      data: {
        name: '[Deleted User]',
        email: anonymousEmail,           // 匿名メールに置換
        phone: null,
        address: null,
        avatarUrl: null,
        deletedAt: new Date(),
        anonymizedAt: new Date(),
        anonymizedId: anonymousId,       // 匿名IDで内部追跡
      },
    });
    affectedTables.push('users');
    recordsAnonymized++;

    // ordersテーブル: 配送先住所をマスク（注文自体は保持）
    const orderUpdate = await tx.order.updateMany({
      where: { userId },
      data: {
        shippingName: '[Deleted User]',
        shippingAddress: null,
        shippingPhone: null,
        // userId は保持（売上集計のため）
        // totalAmount は保持（会計のため）
      },
    });
    affectedTables.push('orders');
    recordsAnonymized += orderUpdate.count;

    // 行動ログは削除（業務上不要）
    await tx.userActivityLog.deleteMany({ where: { userId } });
    affectedTables.push('user_activity_logs');

    // セッション・トークンを全削除
    await tx.refreshToken.deleteMany({ where: { userId } });
    await tx.session.deleteMany({ where: { userId } });

    // メール送信キューから削除（未送信のマーケティングメール）
    await tx.emailQueue.deleteMany({
      where: {
        recipientId: userId,
        type: { in: ['marketing', 'newsletter'] }, // 必須メール（パスワードリセット等）は残す
      },
    });

    // 匿名化履歴を記録（法的証跡）
    await tx.gdprDeletionAudit.create({
      data: {
        userId,
        anonymousId,
        requestedBy,
        affectedTables,
        completedAt: new Date(),
      },
    });
  });

  logger.info({
    userId,
    anonymousId,
    affectedTables,
    recordsAnonymized,
  }, 'User data anonymized');

  return { affectedTables, recordsAnonymized, completedAt: new Date() };
}
```

```typescript
// src/privacy/dataExporter.ts
// データポータビリティ: ユーザーが自分のデータをエクスポート

export async function exportUserData(userId: string): Promise<UserDataExport> {
  const [user, orders, consents] = await Promise.all([
    prisma.user.findUniqueOrThrow({
      where: { id: userId },
      select: {
        id: true,
        name: true,
        email: true,
        phone: true,
        createdAt: true,
      },
    }),
    prisma.order.findMany({
      where: { userId },
      select: {
        id: true,
        createdAt: true,
        totalCents: true,
        status: true,
        items: {
          select: { productName: true, quantity: true, priceCents: true },
        },
      },
      orderBy: { createdAt: 'desc' },
    }),
    prisma.consentRecord.findMany({
      where: { userId },
      orderBy: { grantedAt: 'desc' },
    }),
  ]);

  const exportData: UserDataExport = {
    exportedAt: new Date().toISOString(),
    user: {
      profile: user,
      orders: orders.map(o => ({
        ...o,
        totalAmount: `¥${o.totalCents}`,
      })),
      consentHistory: consents.map(c => ({
        purpose: c.purpose,
        granted: c.granted,
        at: c.grantedAt.toISOString(),
      })),
    },
  };

  return exportData;
}
```

---

## 同意管理

```typescript
// src/privacy/consentManager.ts

type ConsentPurpose = 'essential' | 'analytics' | 'marketing' | 'personalization';

// 同意を記録
export async function recordConsent(
  userId: string,
  purpose: ConsentPurpose,
  granted: boolean,
  ipAddress: string,
  userAgent: string
): Promise<void> {
  await prisma.consentRecord.create({
    data: {
      userId,
      purpose,
      granted,
      grantedAt: new Date(),
      ipAddress,
      userAgent,
      version: process.env.PRIVACY_POLICY_VERSION ?? '1.0', // どのプライバシーポリシーに同意したか
    },
  });

  // マーケティング同意を撤回した場合は即座にキュー削除
  if (purpose === 'marketing' && !granted) {
    await prisma.emailQueue.deleteMany({
      where: { recipientId: userId, type: 'marketing' },
    });
  }
}

// 同意状態を確認
export async function getConsent(userId: string, purpose: ConsentPurpose): Promise<boolean> {
  const latest = await prisma.consentRecord.findFirst({
    where: { userId, purpose },
    orderBy: { grantedAt: 'desc' },
  });

  // essentialは同意不要（サービス提供に必須）
  if (purpose === 'essential') return true;

  return latest?.granted ?? false;
}

// 同意を要求するデコレーター
export function requireConsent(purpose: ConsentPurpose) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const hasConsent = await getConsent(req.user.id, purpose);

    if (!hasConsent) {
      return res.status(403).json({
        error: `User has not consented to ${purpose}`,
        consentUrl: `/privacy/consent?purpose=${purpose}`,
      });
    }

    next();
  };
}
```

---

## まとめ

Claude CodeでGDPRデータ保護を設計する：

1. **CLAUDE.md** にデータカテゴリ分類・匿名化方針・30日以内削除を明記
2. **削除ではなく匿名化** で業務継続（注文履歴・売上集計を保持）
3. **同意を目的ごとに分離** して「マーケティング同意だけ撤回」が可能
4. **匿名化履歴** を法的証跡として保持（どのデータをいつ匿名化したか）

---

*GDPRデータ保護設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
