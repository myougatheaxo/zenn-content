---
title: "Claude Codeで2段階認証（TOTP）を設計する：Google Authenticator・バックアップコード・リカバリー"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "authentication"]
published: true
---

## はじめに

パスワードだけでは不十分——TOTP（Time-based One-Time Password）で2段階認証を実装する。Google AuthenticatorやAuthyで使えるRFC 6238準拠の実装をClaude Codeに設計させる。

---

## CLAUDE.mdに2FA設計ルールを書く

```markdown
## 2段階認証設計ルール

### TOTP実装
- RFC 6238準拠（TOTP）
- シークレットは暗号化してDBに保存（AES-256-GCM）
- QRコードのissuerとaccount名を正確に設定
- 時刻ずれ許容: ±1ステップ（30秒）

### バックアップコード
- 8桁のコードを10個生成
- BCryptでハッシュ化して保存
- 使用済みは即時無効化
- リカバリー後は再生成を強制

### セキュリティ
- TOTP検証はレート制限（5回失敗で15分ロック）
- セットアップ完了まで2FA有効化しない
- 2FA無効化には現在のパスワード+2FAコードを要求
```

---

## 2FA実装の生成

```
TOTPベースの2段階認証を設計してください。

要件：
- セットアップフロー（シークレット生成→QRコード→検証→有効化）
- ログイン時のTOTP検証
- バックアップコード（10個）
- レート制限

生成ファイル: src/auth/twoFactor.ts
```

---

## 生成される2FA実装

```typescript
// src/auth/twoFactor.ts
import { authenticator } from 'otplib';
import QRCode from 'qrcode';
import crypto from 'crypto';
import bcrypt from 'bcrypt';

// TOTPシークレットの暗号化/復号
const ENCRYPTION_KEY = Buffer.from(process.env.TOTP_ENCRYPTION_KEY!, 'hex'); // 32バイト

function encryptSecret(secret: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', ENCRYPTION_KEY, iv);

  const encrypted = Buffer.concat([
    cipher.update(secret, 'utf8'),
    cipher.final(),
  ]);
  const authTag = cipher.getAuthTag();

  // iv:authTag:encrypted を Base64で保存
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted.toString('hex')}`;
}

function decryptSecret(encrypted: string): string {
  const [ivHex, authTagHex, dataHex] = encrypted.split(':');

  const decipher = crypto.createDecipheriv(
    'aes-256-gcm',
    ENCRYPTION_KEY,
    Buffer.from(ivHex, 'hex')
  );
  decipher.setAuthTag(Buffer.from(authTagHex, 'hex'));

  return decipher.update(Buffer.from(dataHex, 'hex')) + decipher.final('utf8');
}

// セットアップ開始: シークレット生成とQRコード返却
export async function setup2FA(userId: string, userEmail: string): Promise<{
  qrCodeDataUrl: string;
  manualEntryKey: string;
}> {
  const secret = authenticator.generateSecret(20); // 20バイト = 32文字のBase32

  // 既存の未確認セットアップを削除
  await prisma.twoFactorSetup.deleteMany({ where: { userId } });

  // 暗号化して一時保存（セットアップ完了まで有効化しない）
  await prisma.twoFactorSetup.create({
    data: {
      userId,
      encryptedSecret: encryptSecret(secret),
      expiresAt: new Date(Date.now() + 10 * 60 * 1000), // 10分で期限切れ
    },
  });

  // otpauth URI生成
  const otpauthUrl = authenticator.keyuri(
    userEmail,
    process.env.APP_NAME ?? 'MyApp',
    secret
  );

  // QRコードをデータURL形式で生成
  const qrCodeDataUrl = await QRCode.toDataURL(otpauthUrl);

  return {
    qrCodeDataUrl,
    manualEntryKey: secret, // Google Authenticatorに手動入力できるように
  };
}

// セットアップ確認: 最初のTOTPコードで有効化
export async function confirm2FA(userId: string, totpCode: string): Promise<{
  backupCodes: string[];
}> {
  const setup = await prisma.twoFactorSetup.findUnique({ where: { userId } });

  if (!setup || setup.expiresAt < new Date()) {
    throw new ValidationError('2FA setup expired. Please restart setup.');
  }

  const secret = decryptSecret(setup.encryptedSecret);

  // TOTPコードを検証（±1ステップの時刻ずれを許容）
  const isValid = authenticator.check(totpCode, secret);
  if (!isValid) {
    throw new ValidationError('Invalid TOTP code');
  }

  // バックアップコードを生成（8桁×10個）
  const plainBackupCodes = Array.from({ length: 10 }, () =>
    crypto.randomBytes(4).toString('hex').toUpperCase() // 8桁の16進数
  );

  // バックアップコードをBCryptでハッシュ化
  const hashedBackupCodes = await Promise.all(
    plainBackupCodes.map(code => bcrypt.hash(code, 10))
  );

  // トランザクションで2FA有効化
  await prisma.$transaction(async (tx) => {
    // セットアップを削除
    await tx.twoFactorSetup.delete({ where: { userId } });

    // 本番テーブルに移行
    await tx.twoFactor.upsert({
      where: { userId },
      create: {
        userId,
        encryptedSecret: setup.encryptedSecret,
        enabledAt: new Date(),
      },
      update: {
        encryptedSecret: setup.encryptedSecret,
        enabledAt: new Date(),
      },
    });

    // バックアップコードを保存
    await tx.backupCode.deleteMany({ where: { userId } });
    await tx.backupCode.createMany({
      data: hashedBackupCodes.map(hash => ({
        userId,
        codeHash: hash,
        usedAt: null,
      })),
    });

    // ユーザーの2FA有効フラグを更新
    await tx.user.update({
      where: { id: userId },
      data: { twoFactorEnabled: true },
    });
  });

  return { backupCodes: plainBackupCodes }; // 一度だけ平文で返す
}

// ログイン時のTOTP検証
export async function verify2FA(
  userId: string,
  code: string
): Promise<void> {
  // レート制限チェック
  const rateKey = `2fa:attempts:${userId}`;
  const attempts = await redis.incr(rateKey);
  await redis.expire(rateKey, 900); // 15分ウィンドウ

  if (attempts > 5) {
    throw new TooManyRequestsError('Too many failed 2FA attempts. Try again in 15 minutes.');
  }

  const twoFactor = await prisma.twoFactor.findUnique({ where: { userId } });
  if (!twoFactor) throw new ValidationError('2FA not enabled');

  const secret = decryptSecret(twoFactor.encryptedSecret);

  // TOTPコード検証
  if (authenticator.check(code, secret)) {
    await redis.del(rateKey); // 成功したらカウントリセット
    return;
  }

  // バックアップコードを試す
  const backupCodes = await prisma.backupCode.findMany({
    where: { userId, usedAt: null }, // 未使用のみ
  });

  for (const bc of backupCodes) {
    if (await bcrypt.compare(code, bc.codeHash)) {
      // バックアップコードを使用済みにマーク
      await prisma.backupCode.update({
        where: { id: bc.id },
        data: { usedAt: new Date() },
      });

      await redis.del(rateKey);

      // 残りのバックアップコードが少ない場合に警告
      const remainingCount = backupCodes.length - 1;
      if (remainingCount <= 2) {
        await sendNotification(userId, 'backup_codes_running_low', { remainingCount });
      }

      return;
    }
  }

  throw new ValidationError('Invalid 2FA code');
}
```

---

## ログインフローへの組み込み

```typescript
// src/routes/auth.ts

// Step 1: パスワード認証
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await verifyPassword(email, password);

  if (user.twoFactorEnabled) {
    // 2FAが必要: 一時トークンを発行してフロントエンドに通知
    const tempToken = await createTempToken(user.id, '2fa_pending');
    return res.json({
      requiresTwoFactor: true,
      tempToken,
    });
  }

  // 2FA不要: 通常のJWTを発行
  const tokens = await generateTokenPair(user.id);
  res.json(tokens);
});

// Step 2: 2FAコード検証
router.post('/login/2fa', async (req, res) => {
  const { tempToken, code } = req.body;

  // 一時トークンを検証
  const userId = await verifyTempToken(tempToken, '2fa_pending');
  if (!userId) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }

  // TOTPコードを検証
  await verify2FA(userId, code);

  // 本番JWTを発行
  const tokens = await generateTokenPair(userId);
  res.json(tokens);
});
```

---

## まとめ

Claude Codeで2段階認証（TOTP）を設計する：

1. **CLAUDE.md** にRFC 6238・シークレット暗号化・バックアップコード10個・レート制限5回を明記
2. **AES-256-GCM** でTOTPシークレットを暗号化してDB保存（平文禁止）
3. **セットアップ確認フロー** で最初のTOTPコードを検証してから有効化
4. **バックアップコード** をBCryptでハッシュ化・使用済みは即時無効化

---

*2FA設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
