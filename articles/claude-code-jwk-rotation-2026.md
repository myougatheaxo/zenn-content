---
title: "Claude CodeでJWTキーローテーションを設計する：JWK・kid管理・ゼロダウンタイム更新"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "oauth"]
published: true
published_at: "2026-03-14 14:00"
---

## はじめに

JWTの署名キーを変更すると既存トークンが全て無効になる——JWK（JSON Web Key）セットで複数キーを管理し、`kid`（Key ID）で切り替えることでゼロダウンタイムのキーローテーションを実現する。Claude Codeに設計させる。

---

## CLAUDE.mdにJWKローテーション設計ルールを書く

```markdown
## JWTキーローテーション設計ルール

### キー管理
- 2つのキーを同時保持: current（署名用）+ previous（検証のみ）
- kidにバージョン番号またはタイムスタンプを付与
- キー素材はAWS KMS / Vault から取得（ローカルファイル禁止）

### ローテーション手順（ゼロダウンタイム）
1. 新キーを生成・KMSに保存
2. JWKSエンドポイントに新キーを追加（current）
3. 古いキーをpreviousとして継続公開（既存トークン検証用）
4. access_tokenの最大TTL（15分）経過後に古いキーを削除

### JWKS公開エンドポイント
- GET /.well-known/jwks.json で公開鍵セットを公開
- Cache-Control: max-age=3600（1時間キャッシュ）
- kid でリクエストされたキーを選択して署名
```

---

## JWKローテーション実装の生成

```
JWT JWKキーローテーションシステムを設計してください。

要件：
- 複数キー（current/previous）管理
- JWKSエンドポイント
- ゼロダウンタイムローテーション
- kid付きJWT署名・検証

生成ファイル: src/auth/jwk/
```

---

## 生成されるJWKローテーション実装

```typescript
// src/auth/jwk/keyManager.ts — JWKキー管理

import { generateKeyPair, exportJWK, importJWK, JWK } from 'jose';

interface JWKEntry {
  kid: string;
  keyPair: { privateKey: KeyObject; publicKey: KeyObject };
  createdAt: Date;
  status: 'current' | 'previous' | 'expired';
}

export class JWKKeyManager {
  private keys: Map<string, JWKEntry> = new Map();

  async initialize(): Promise<void> {
    // Redisから既存キーを読み込み
    const storedKeys = await redis.get('jwk:keys');
    if (storedKeys) {
      const parsed = JSON.parse(storedKeys);
      for (const entry of parsed) {
        const privateKey = await importJWK(entry.privateJwk, 'RS256');
        const publicKey = await importJWK(entry.publicJwk, 'RS256');
        this.keys.set(entry.kid, {
          kid: entry.kid,
          keyPair: { privateKey, publicKey } as any,
          createdAt: new Date(entry.createdAt),
          status: entry.status,
        });
      }
      logger.info({ keyCount: this.keys.size }, 'JWK keys loaded from Redis');
    } else {
      // 初回起動: 最初のキーを生成
      await this.generateNewKey();
    }
  }

  // 新しいキーペアを生成してcurrentに設定
  async generateNewKey(): Promise<string> {
    const kid = `key-${Date.now()}`;
    const { privateKey, publicKey } = await generateKeyPair('RS256', {
      modulusLength: 2048,
    });

    // 既存のcurrentをpreviousに降格
    for (const [, entry] of this.keys) {
      if (entry.status === 'current') {
        entry.status = 'previous';
      }
    }

    this.keys.set(kid, {
      kid,
      keyPair: { privateKey, publicKey } as any,
      createdAt: new Date(),
      status: 'current',
    });

    await this.persistKeys();
    logger.info({ kid }, 'New JWK key generated');
    return kid;
  }

  // 現在のキー（署名用）を取得
  getCurrentKey(): JWKEntry {
    for (const [, entry] of this.keys) {
      if (entry.status === 'current') return entry;
    }
    throw new Error('No current key available');
  }

  // kidでキーを検索（検証用）
  getKeyById(kid: string): JWKEntry | undefined {
    return this.keys.get(kid);
  }

  // JWKS形式で全公開鍵を返す
  async getPublicJWKS(): Promise<{ keys: JWK[] }> {
    const jwks: JWK[] = [];
    for (const [, entry] of this.keys) {
      if (entry.status === 'expired') continue; // 期限切れは公開しない
      const publicJwk = await exportJWK(entry.keyPair.publicKey);
      jwks.push({
        ...publicJwk,
        kid: entry.kid,
        use: 'sig',
        alg: 'RS256',
      });
    }
    return { keys: jwks };
  }

  // previousキーを削除（ローテーション完了後）
  async expirePreviousKeys(): Promise<void> {
    let expired = 0;
    for (const [kid, entry] of this.keys) {
      if (entry.status === 'previous') {
        // access_tokenの最大TTL以上経過していれば削除
        const ageMs = Date.now() - entry.createdAt.getTime();
        if (ageMs > 30 * 60 * 1000) { // 30分（access_token 15分の2倍）
          entry.status = 'expired';
          expired++;
          logger.info({ kid }, 'JWK key expired');
        }
      }
    }
    if (expired > 0) await this.persistKeys();
  }

  private async persistKeys(): Promise<void> {
    const toStore = await Promise.all(
      [...this.keys.values()].map(async entry => ({
        kid: entry.kid,
        privateJwk: await exportJWK(entry.keyPair.privateKey),
        publicJwk: await exportJWK(entry.keyPair.publicKey),
        createdAt: entry.createdAt.toISOString(),
        status: entry.status,
      }))
    );
    await redis.set('jwk:keys', JSON.stringify(toStore), { EX: 86400 * 30 });
  }
}

export const keyManager = new JWKKeyManager();
```

```typescript
// src/auth/jwk/jwt.ts — kid付きJWT署名・検証

import { SignJWT, jwtVerify, decodeProtectedHeader } from 'jose';

export async function signToken(
  payload: Record<string, unknown>,
  expiresIn = '15m'
): Promise<string> {
  const currentKey = keyManager.getCurrentKey();

  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'RS256', kid: currentKey.kid })
    .setIssuedAt()
    .setIssuer(process.env.JWT_ISSUER!)
    .setAudience(process.env.JWT_AUDIENCE!)
    .setExpirationTime(expiresIn)
    .sign(currentKey.keyPair.privateKey);
}

export async function verifyToken(token: string): Promise<JWTPayload> {
  // ヘッダーからkidを取得（署名前に確認）
  const header = decodeProtectedHeader(token);
  const kid = header.kid;

  if (!kid) {
    throw new JWTError('Token missing kid header');
  }

  // kidに対応するキーを検索
  const keyEntry = keyManager.getKeyById(kid);
  if (!keyEntry) {
    throw new JWTError(`Unknown key ID: ${kid}`);
  }
  if (keyEntry.status === 'expired') {
    throw new JWTError(`Key ${kid} has expired`);
  }

  try {
    const { payload } = await jwtVerify(token, keyEntry.keyPair.publicKey, {
      issuer: process.env.JWT_ISSUER,
      audience: process.env.JWT_AUDIENCE,
    });
    return payload;
  } catch (e) {
    throw new JWTError(`Token verification failed: ${(e as Error).message}`);
  }
}
```

```typescript
// src/auth/jwk/routes.ts — JWKSエンドポイント + ローテーションAPI

// GET /.well-known/jwks.json — OAuth標準エンドポイント
router.get('/.well-known/jwks.json', async (req, res) => {
  const jwks = await keyManager.getPublicJWKS();
  res.setHeader('Cache-Control', 'public, max-age=3600'); // 1時間キャッシュ
  res.json(jwks);
});

// POST /admin/jwk/rotate — キーローテーション（管理者のみ）
router.post('/admin/jwk/rotate', requireAdmin, async (req, res) => {
  const newKid = await keyManager.generateNewKey();

  logger.info({ newKid }, 'JWK key rotation initiated');
  await sendSlackAlert(`🔑 JWT Key Rotated: New key ${newKid} is now active`);

  res.json({
    message: 'Key rotation initiated',
    newKid,
    note: 'Previous key will expire in 30 minutes after all access tokens expire',
  });
});

// CronJob: 毎時 previousキーの期限切れチェック
// schedule: '0 * * * *'
export async function cleanupExpiredKeys(): Promise<void> {
  await keyManager.expirePreviousKeys();
}
```

---

## まとめ

Claude CodeでJWTキーローテーションを設計する：

1. **CLAUDE.md** に2キー同時保持（current/previous）・kidによる選択・30分経過後にprevious削除を明記
2. **kidヘッダー** でどのキーで署名されたかを判別——ローテーション中も旧トークンを検証可能
3. **JWKS公開エンドポイント** でOAuth/OIDC準拠の公開鍵配布——SPAや他サービスが動的に鍵取得
4. **expirePreviousKeys()** でaccess_token TTL（15分）×2=30分後に古いキーを自動削除

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
