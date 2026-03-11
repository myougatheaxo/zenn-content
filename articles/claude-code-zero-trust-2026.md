---
title: "Claude CodeでZero Trustセキュリティを設計する：mTLS・サービスメッシュ・最小権限"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "security", "kubernetes", "typescript", "devops"]
published: true
published_at: "2026-03-12 09:00"
---

## はじめに

「社内ネットワークだから安全」は過去の話——Zero Trustでは全リクエストを検証する。mTLS・サービスアカウント・最小権限アクセスをClaude Codeに設計させる。

---

## CLAUDE.mdにZero Trust設計ルールを書く

```markdown
## Zero Trustセキュリティ設計ルール

### 原則
- Never Trust, Always Verify（VPNの中でも検証）
- 最小権限（マイクロサービス間も必要なエンドポイントのみ）
- 全通信を暗号化（mTLS、サービス間もTLS必須）

### 認証・認可
- サービス間: SPIFFE/SPIRE or Istio mTLS
- ユーザー→サービス: JWTのAudience検証（サービスごとに別Audience）
- API Keys: ローテーション必須（90日）

### 監査
- 全APIアクセスをログに記録（ユーザー/サービス/エンドポイント/結果）
- 異常検知: 同一IPから5分で100リクエスト以上
- セキュリティイベントはSIEMに即時転送
```

---

## Zero Trustアーキテクチャの生成

```
Zero Trustセキュリティアーキテクチャを設計してください。

要件：
- mTLSサービス間通信
- JWT Audience検証
- APIキー管理（ローテーション）
- 異常検知
- 監査ログ

生成ファイル: src/security/zero-trust/
```

---

## 生成されるZero Trust実装

```typescript
// src/security/zero-trust/jwtValidator.ts

// サービスごとにAudienceを分離（権限の分離）
const SERVICE_AUDIENCES: Record<string, string> = {
  'api-gateway':     'api:gateway',
  'order-service':   'api:orders',
  'payment-service': 'api:payments',
  'user-service':    'api:users',
};

// JWTのAudience検証（サービス間も含む）
export async function validateJWT(
  token: string,
  requiredAudience: string
): Promise<JWTPayload> {
  // JWKSから公開鍵を取得（キャッシュ付き）
  const keys = await getJWKS();

  const { payload } = await jwtVerify(token, keys, {
    issuer: process.env.JWT_ISSUER!,
    audience: requiredAudience,
    algorithms: ['RS256'],
    clockTolerance: 30, // 時刻ずれ30秒まで許容
  });

  return payload;
}

// サービス間通信のトークン生成（M2M）
export async function createServiceToken(
  fromService: string,
  toService: string,
  scopes: string[]
): Promise<string> {
  const targetAudience = SERVICE_AUDIENCES[toService];
  if (!targetAudience) throw new Error(`Unknown service: ${toService}`);

  const privateKey = await getServicePrivateKey(fromService);

  return new SignJWT({
    sub: `service:${fromService}`,
    scope: scopes.join(' '),
    service_name: fromService,
  })
    .setProtectedHeader({ alg: 'RS256' })
    .setIssuedAt()
    .setExpirationTime('5m') // サービス間トークンは短期間（5分）
    .setAudience(targetAudience)
    .setIssuer(process.env.JWT_ISSUER!)
    .sign(privateKey);
}
```

```typescript
// src/security/zero-trust/apiKeyManager.ts
// APIキー管理（ローテーション・最小権限）

import { createHash, randomBytes } from 'crypto';

// APIキー構造: prefix_randomhex（プレフィックスでサービスを識別）
export async function generateApiKey(params: {
  userId: string;
  name: string;        // "Production Server" など識別名
  scopes: string[];    // ['orders:read', 'orders:write'] など最小権限
  expiresAt?: Date;    // 90日後を推奨
}): Promise<{ key: string; id: string }> {
  const rawKey = `mk_${params.userId.slice(0, 8)}_${randomBytes(24).toString('hex')}`;

  // ハッシュして保存（平文は一度だけ返す）
  const keyHash = createHash('sha256').update(rawKey).digest('hex');

  const record = await prisma.apiKey.create({
    data: {
      userId: params.userId,
      name: params.name,
      keyHash,
      keyPrefix: rawKey.slice(0, 16), // 管理UI表示用（フル値は非表示）
      scopes: params.scopes,
      expiresAt: params.expiresAt ?? new Date(Date.now() + 90 * 24 * 60 * 60 * 1000),
    },
  });

  logger.info({ userId: params.userId, keyId: record.id, scopes: params.scopes }, 'API key created');

  return { key: rawKey, id: record.id }; // keyは一度だけ返す
}

// APIキーの検証
export async function validateApiKey(
  rawKey: string,
  requiredScope: string
): Promise<ApiKeyRecord> {
  const keyHash = createHash('sha256').update(rawKey).digest('hex');

  const record = await prisma.apiKey.findFirst({
    where: {
      keyHash,
      revokedAt: null,
      expiresAt: { gt: new Date() },
    },
  });

  if (!record) throw new UnauthorizedError('Invalid or expired API key');

  // スコープチェック
  if (!record.scopes.includes(requiredScope)) {
    throw new ForbiddenError(`Missing scope: ${requiredScope}`);
  }

  // 最終使用日時を更新（非同期）
  prisma.apiKey.update({
    where: { id: record.id },
    data: { lastUsedAt: new Date() },
  }).catch(() => {}); // ベストエフォート

  return record;
}

// 90日期限切れが近いキーの自動通知
export async function notifyExpiringKeys(): Promise<void> {
  const soon = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000); // 7日後

  const expiring = await prisma.apiKey.findMany({
    where: {
      expiresAt: { lte: soon, gt: new Date() },
      revokedAt: null,
    },
    include: { user: true },
  });

  for (const key of expiring) {
    await sendNotification(key.userId, 'api_key_expiring', {
      keyName: key.name,
      expiresAt: key.expiresAt,
    });
  }
}
```

```typescript
// src/security/zero-trust/anomalyDetector.ts

// レート制限 + 異常検知
export async function detectAnomaly(params: {
  identifier: string;  // userId or IP
  endpoint: string;
  method: string;
}): Promise<void> {
  const windowKey = `anomaly:${params.identifier}:${Math.floor(Date.now() / 300_000)}`; // 5分ウィンドウ

  const count = await redis.incr(windowKey);
  await redis.expire(windowKey, 300);

  // 5分間に100リクエスト以上 = 異常
  if (count === 100) {
    logger.warn({
      identifier: params.identifier,
      endpoint: params.endpoint,
      count,
    }, 'Anomaly detected: high request rate');

    // SIEMにセキュリティイベントを送信
    await sendSIEMEvent({
      type: 'RATE_ANOMALY',
      severity: 'HIGH',
      identifier: params.identifier,
      endpoint: params.endpoint,
      requestCount: count,
      windowMinutes: 5,
    });
  }

  // 500以上は自動ブロック（DDoS対策）
  if (count >= 500) {
    await redis.set(`blocked:${params.identifier}`, '1', { EX: 3600 }); // 1時間ブロック
    throw new TooManyRequestsError('Temporarily blocked due to suspicious activity');
  }
}
```

---

## まとめ

Claude CodeでZero Trustを設計する：

1. **CLAUDE.md** にNever Trust Always Verify・mTLS必須・JWT Audience検証・90日APIキーを明記
2. **JWT Audience をサービスごとに分離** してトークンの横流し攻撃を防ぐ
3. **APIキーはsha256ハッシュで保存** し平文は一度だけ返す（漏洩時の影響範囲を最小化）
4. **5分100リクエスト超過** で自動SIEMアラート、500超過で1時間自動ブロック

---

*Zero Trust設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
