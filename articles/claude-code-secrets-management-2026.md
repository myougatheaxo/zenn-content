---
title: "Claude Codeでシークレット管理を設計する：AWS Secrets ManagerとZodバリデーション"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "aws", "security"]
published: true
---

## はじめに

`.env`ファイルをGitにコミットするのはアウト。ローカルに秘密情報をハードコードするのもアウト。Claude Codeにシークレット管理の安全な設計を生成させる。

---

## CLAUDE.mdにシークレット管理ルールを書く

```markdown
## シークレット管理ルール

### 禁止事項（必須）
- 秘密情報（APIキー、DB接続情報、JWT秘密鍵）をコードにハードコード禁止
- .env ファイルをGitリポジトリにコミット禁止
- ログに秘密情報を出力禁止

### 本番環境
- AWS Secrets Manager または HashiCorp Vault を使用
- アプリ起動時にシークレットをメモリにロード（ファイルに書き出さない）
- 30日以内にローテーション（自動ローテーション設定）

### 開発環境
- .env.example に変数名のみ記載（値は書かない）
- 実際の値は 1Password / Bitwarden で管理
- ローカル用ダミー値はコメントで .env.example に記載

### バリデーション
- 起動時に必須シークレットが全て揃っているか確認
- Zodスキーマで型・形式を検証
- 1つでも欠けたらプロセスを落とす（早期失敗原則）
```

---

## シークレット管理の生成

```
AWS Secrets Managerを使ったシークレット管理を設計してください。

要件：
- 起動時にSecrets ManagerからシークレットをロードZ
- Zodで型バリデーション
- 必須シークレットが欠けたら起動失敗
- ローカル開発は.envフォールバック
- シークレットをキャッシュ（毎リクエストで取得しない）

生成ファイル: src/config/secrets.ts
```

---

## 生成されるシークレット管理

```typescript
// src/config/secrets.ts
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
import { z } from 'zod';

const secretsManager = new SecretsManagerClient({ region: process.env.AWS_REGION });

// 全シークレットのZodスキーマ
const secretsSchema = z.object({
  DATABASE_URL: z.string().url('DATABASE_URL must be a valid URL'),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_', 'Invalid Stripe secret key format'),
  REDIS_URL: z.string().url(),
  AWS_SES_REGION: z.string().optional(),
  GITHUB_WEBHOOK_SECRET: z.string().min(16),
});

export type Secrets = z.infer<typeof secretsSchema>;

// シークレットをキャッシュ（起動時に1回だけロード）
let cachedSecrets: Secrets | null = null;

async function loadFromSecretsManager(secretName: string): Promise<Record<string, string>> {
  const command = new GetSecretValueCommand({ SecretId: secretName });
  const response = await secretsManager.send(command);

  if (!response.SecretString) {
    throw new Error(`Secret ${secretName} has no string value`);
  }

  return JSON.parse(response.SecretString);
}

export async function loadSecrets(): Promise<Secrets> {
  if (cachedSecrets) return cachedSecrets;

  let rawSecrets: Record<string, unknown>;

  if (process.env.NODE_ENV === 'production') {
    // 本番: Secrets Managerからロード
    const secretName = process.env.SECRET_NAME ?? 'myapp/production';
    rawSecrets = await loadFromSecretsManager(secretName);
    logger.info({ secretName }, 'Secrets loaded from Secrets Manager');
  } else {
    // 開発: .envファイルからロード（dotenvが設定済みの前提）
    rawSecrets = process.env;
    logger.info('Secrets loaded from .env file');
  }

  // Zodで型バリデーション
  const result = secretsSchema.safeParse(rawSecrets);

  if (!result.success) {
    const missing = result.error.errors.map(e => `${e.path.join('.')}: ${e.message}`);
    logger.fatal({ missing }, 'Secret validation failed — refusing to start');
    process.exit(1); // 起動失敗（早期失敗原則）
  }

  cachedSecrets = result.data;
  return cachedSecrets;
}

export function getSecrets(): Secrets {
  if (!cachedSecrets) {
    throw new Error('Secrets not loaded — call loadSecrets() first in app startup');
  }
  return cachedSecrets;
}
```

```typescript
// src/main.ts
// アプリ起動時にシークレットをロード
async function bootstrap(): Promise<void> {
  await loadSecrets(); // 失敗したらプロセス終了

  const secrets = getSecrets();
  await connectDatabase(secrets.DATABASE_URL);
  await connectRedis(secrets.REDIS_URL);

  const app = createApp(secrets);
  app.listen(3000, () => logger.info('Server started on port 3000'));
}

bootstrap().catch(err => {
  logger.fatal({ err }, 'Failed to start application');
  process.exit(1);
});
```

---

## .env.example テンプレート

```bash
# .env.example — 変数名のみ記載（値は記載しない）
# 実際の値は 1Password の "myapp-development" アイテムを参照

# データベース
DATABASE_URL=postgresql://user:password@localhost:5432/myapp

# JWT
JWT_SECRET=<32文字以上のランダム文字列: openssl rand -base64 32 で生成>

# Stripe
STRIPE_SECRET_KEY=sk_test_...

# Redis
REDIS_URL=redis://localhost:6379

# GitHub Webhook
GITHUB_WEBHOOK_SECRET=<16文字以上のランダム文字列>
```

---

## まとめ

Claude Codeでシークレット管理を設計する：

1. **CLAUDE.md** にハードコード禁止・.envをGit禁止・起動バリデーション必須を明記
2. **Secrets Manager** で本番シークレットを安全に管理
3. **Zodバリデーション** で起動時に全シークレットの存在・形式を確認
4. **早期失敗** で不完全な設定のままサービスが動かないようにする

---

*シークレット管理の問題（ハードコード、.envのGitコミット）を検出するスキルは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
