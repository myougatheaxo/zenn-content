---
title: "Claude CodeでWebAuthn・パスキーを設計する：パスワードレス認証・生体認証・FIDO2"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "authentication"]
published: true
---

## はじめに

パスワードは時代遅れ——WebAuthn（FIDO2）でパスキー認証を実装する。TouchID・FaceID・Windows Helloで「パスワード不要」のセキュアな認証をClaude Codeに設計させる。

---

## CLAUDE.mdにWebAuthn設計ルールを書く

```markdown
## WebAuthn/パスキー設計ルール

### 登録フロー
- チャレンジは毎回ランダム生成（32バイト）
- チャレンジはサーバーセッションに保存（期限付き）
- attestationは"none"（デバイス証明書不要）
- 既存のパスキーを確認してから登録（重複防止）

### 認証フロー
- チャレンジは毎回ランダム生成（32バイト）
- rpID: 本番ドメイン固定（フィッシング防止）
- userVerification: "required"（生体認証強制）
- signCount検証でリプレイ攻撃を防ぐ

### セキュリティ
- 公開鍵はDBに保存（秘密鍵はデバイス内で管理）
- signCountが戻った場合はクレデンシャル無効化（クローン検出）
- バックアップとしてTOTP 2FAも提供
```

---

## WebAuthn実装の生成

```
WebAuthn（FIDO2）パスキー認証を設計してください。

要件：
- 登録フロー（デバイス登録）
- 認証フロー（パスキーでログイン）
- 複数デバイス管理
- フォールバック（パスワード認証）

生成ファイル: src/auth/webauthn.ts
```

---

## 生成されるWebAuthn実装

```typescript
// src/auth/webauthn.ts
import {
  generateRegistrationOptions,
  verifyRegistrationResponse,
  generateAuthenticationOptions,
  verifyAuthenticationResponse,
  type RegistrationResponseJSON,
  type AuthenticationResponseJSON,
} from '@simplewebauthn/server';

const RP_NAME = process.env.APP_NAME ?? 'MyApp';
const RP_ID = process.env.RP_ID ?? 'localhost';  // 本番: 'myapp.com'
const ORIGIN = process.env.ORIGIN ?? 'http://localhost:3000';

// ============================
// 登録フロー
// ============================

// Step 1: 登録オプションを生成してフロントエンドに返す
export async function startRegistration(userId: string): Promise<PublicKeyCredentialCreationOptionsJSON> {
  const user = await prisma.user.findUniqueOrThrow({ where: { id: userId } });

  // 既存のパスキーを取得（同じデバイスの重複登録を防ぐ）
  const existingPasskeys = await prisma.passkey.findMany({
    where: { userId },
    select: { credentialId: true, transports: true },
  });

  const options = await generateRegistrationOptions({
    rpName: RP_NAME,
    rpID: RP_ID,
    userID: Buffer.from(userId),
    userName: user.email,
    userDisplayName: user.name,
    // 既存デバイスは再登録できないようにする
    excludeCredentials: existingPasskeys.map(pk => ({
      id: Buffer.from(pk.credentialId, 'base64'),
      type: 'public-key',
      transports: pk.transports as AuthenticatorTransport[],
    })),
    authenticatorSelection: {
      residentKey: 'preferred',     // パスキー（discoverable credential）推奨
      userVerification: 'required', // 生体認証強制
    },
    attestationType: 'none', // 証明書不要（ほとんどのユースケース）
    timeout: 60_000,
  });

  // チャレンジをセッションに保存（検証時に使用）
  await redis.set(
    `webauthn:reg:challenge:${userId}`,
    options.challenge,
    { EX: 60 } // 1分で期限切れ
  );

  return options;
}

// Step 2: 登録レスポンスを検証してパスキーを保存
export async function finishRegistration(
  userId: string,
  credential: RegistrationResponseJSON,
  deviceName: string
): Promise<void> {
  const expectedChallenge = await redis.get(`webauthn:reg:challenge:${userId}`);
  if (!expectedChallenge) {
    throw new ValidationError('Registration challenge expired. Please try again.');
  }

  const verification = await verifyRegistrationResponse({
    response: credential,
    expectedChallenge,
    expectedOrigin: ORIGIN,
    expectedRPID: RP_ID,
    requireUserVerification: true,
  });

  if (!verification.verified || !verification.registrationInfo) {
    throw new ValidationError('Passkey registration failed. Please try again.');
  }

  const { credentialPublicKey, credentialID, counter, credentialDeviceType } =
    verification.registrationInfo;

  // パスキーをDBに保存
  await prisma.passkey.create({
    data: {
      userId,
      credentialId: Buffer.from(credentialID).toString('base64'),
      publicKey: Buffer.from(credentialPublicKey).toString('base64'),
      counter,                // リプレイ攻撃防止用カウンター
      deviceType: credentialDeviceType, // 'singleDevice' | 'multiDevice'
      deviceName,             // ユーザー識別用（"iPhone 15", "MacBook Pro"等）
      transports: credential.response.transports ?? [],
    },
  });

  // チャレンジを削除
  await redis.del(`webauthn:reg:challenge:${userId}`);

  logger.info({ userId, deviceName }, 'Passkey registered');
}

// ============================
// 認証フロー
// ============================

// Step 1: 認証オプションを生成
export async function startAuthentication(
  email?: string  // メール指定なし = パスキーの自動検出（discoverable credential）
): Promise<PublicKeyCredentialRequestOptionsJSON> {
  let allowCredentials: PublicKeyCredentialDescriptorJSON[] = [];

  if (email) {
    // 特定ユーザーのパスキーのみ使用
    const user = await prisma.user.findUnique({ where: { email } });
    if (user) {
      const passkeys = await prisma.passkey.findMany({ where: { userId: user.id } });
      allowCredentials = passkeys.map(pk => ({
        id: pk.credentialId,
        type: 'public-key',
        transports: pk.transports as AuthenticatorTransport[],
      }));
    }
  }
  // email未指定 + allowCredentials空 = ブラウザがパスキーを自動提示

  const options = await generateAuthenticationOptions({
    rpID: RP_ID,
    allowCredentials,
    userVerification: 'required',
    timeout: 60_000,
  });

  // セッションIDでチャレンジを保存（ユーザー不特定のため）
  const sessionId = crypto.randomUUID();
  await redis.set(`webauthn:auth:challenge:${sessionId}`, options.challenge, { EX: 60 });

  return { ...options, sessionId } as any;
}

// Step 2: 認証レスポンスを検証してJWTを発行
export async function finishAuthentication(
  sessionId: string,
  credential: AuthenticationResponseJSON
): Promise<{ userId: string }> {
  const expectedChallenge = await redis.get(`webauthn:auth:challenge:${sessionId}`);
  if (!expectedChallenge) {
    throw new ValidationError('Authentication challenge expired. Please try again.');
  }

  // credentialIDからパスキーを取得
  const passkey = await prisma.passkey.findFirst({
    where: { credentialId: credential.id },
    include: { user: true },
  });

  if (!passkey) {
    throw new ValidationError('Unknown passkey');
  }

  const verification = await verifyAuthenticationResponse({
    response: credential,
    expectedChallenge,
    expectedOrigin: ORIGIN,
    expectedRPID: RP_ID,
    authenticator: {
      credentialPublicKey: Buffer.from(passkey.publicKey, 'base64'),
      credentialID: Buffer.from(passkey.credentialId, 'base64'),
      counter: passkey.counter,
      transports: passkey.transports as AuthenticatorTransport[],
    },
    requireUserVerification: true,
  });

  if (!verification.verified) {
    throw new ValidationError('Passkey authentication failed');
  }

  const { authenticationInfo } = verification;

  // signCountが戻った場合 = クレデンシャルクローンの可能性
  if (authenticationInfo.newCounter < passkey.counter) {
    // パスキーを無効化してユーザーに通知
    await prisma.passkey.update({
      where: { id: passkey.id },
      data: { revokedAt: new Date(), revokedReason: 'counter_regression' },
    });
    await sendNotification(passkey.userId, 'passkey_cloned_detected', {});
    throw new SecurityError('Passkey may be cloned. Please re-register.');
  }

  // signCountを更新
  await prisma.passkey.update({
    where: { id: passkey.id },
    data: {
      counter: authenticationInfo.newCounter,
      lastUsedAt: new Date(),
    },
  });

  await redis.del(`webauthn:auth:challenge:${sessionId}`);

  return { userId: passkey.userId };
}
```

---

## APIルートとフロントエンドフロー

```typescript
// src/routes/webauthn.ts

// 登録開始
router.post('/webauthn/register/start', requireAuth, async (req, res) => {
  const options = await startRegistration(req.user.id);
  res.json(options);
});

// 登録完了
router.post('/webauthn/register/finish', requireAuth, async (req, res) => {
  const { credential, deviceName } = req.body;
  await finishRegistration(req.user.id, credential, deviceName);
  res.json({ success: true });
});

// 認証開始（ログイン）
router.post('/webauthn/auth/start', async (req, res) => {
  const { email } = req.body;
  const options = await startAuthentication(email);
  res.json(options);
});

// 認証完了
router.post('/webauthn/auth/finish', async (req, res) => {
  const { sessionId, credential } = req.body;
  const { userId } = await finishAuthentication(sessionId, credential);
  const tokens = await generateTokenPair(userId);
  res.json(tokens);
});
```

```typescript
// frontend: パスキー登録（React）
import { startRegistration } from '@simplewebauthn/browser';

async function registerPasskey(deviceName: string) {
  // サーバーからオプションを取得
  const optionsRes = await fetch('/api/webauthn/register/start', { method: 'POST' });
  const options = await optionsRes.json();

  // ブラウザのWebAuthn APIを呼び出し
  const credential = await startRegistration(options);

  // サーバーに検証を依頼
  await fetch('/api/webauthn/register/finish', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ credential, deviceName }),
  });
}
```

---

## まとめ

Claude CodeでWebAuthn/パスキーを設計する：

1. **CLAUDE.md** にチャレンジ毎回生成・userVerification required・signCount検証を明記
2. **チャレンジはRedisに1分間保存** して有効期限後は無効（リプレイ攻撃防止）
3. **signCountが戻った場合** はクレデンシャルクローンを検出して即座に無効化
4. **discoverable credential** でメール入力不要のパスキー自動提示UXを実現

---

*WebAuthn設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
