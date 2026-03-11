---
title: "Claude Codeで環境変数管理を設計する：型安全な設定とシークレット保護"
emoji: "⚙️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "環境変数", "セキュリティ"]
published: true
---

## はじめに

`process.env.DATABASE_URL` を直接コードに散らばらせると、スペルミスで本番がクラッシュしたり、必須設定の確認を忘れたりする。Claude Codeに型安全な設定管理を設計させる。

---

## CLAUDE.mdに環境変数管理ルールを書く

```markdown
## 環境変数管理ルール

### アクセス方法
- `process.env.*` を直接使わない
- 全ての環境変数は `src/config/env.ts` 経由でアクセス
- envモジュールをアプリ起動時に検証（必須変数が欠けたらクラッシュ）

### 型安全
- Zodで環境変数スキーマを定義
- 型変換: 文字列 → 数値/真偽値/URLは自動変換
- デフォルト値は明示的に定義

### シークレット管理
- 機密情報（APIキー・DBパスワード）は .env に書く（絶対にコミットしない）
- .env.example を常に最新に保つ（値を含めない、キーのみ）
- CI/CDシークレットはプラットフォームの環境変数機能を使う

### 環境区分
- development: .env.local
- test: .env.test
- production: 環境変数（.envファイル不使用）

### 禁止
- .env ファイルをコミットする
- `process.env.NODE_ENV === 'production'` 以外でのenv分岐
- 環境変数のデフォルト値に本物のシークレットを使う
```

---

## 型安全なenvモジュールの生成

```
Zodを使った型安全な環境変数モジュールを生成してください。

環境変数リスト:
- DATABASE_URL: string（有効なPostgres URL）
- REDIS_URL: string（有効なRedis URL）
- JWT_SECRET: string（最低32文字）
- PORT: number（default: 3000）
- NODE_ENV: 'development' | 'test' | 'production'
- LOG_LEVEL: 'debug' | 'info' | 'warn' | 'error'（default: 'info'）
- SENDGRID_API_KEY: string（オプション、メール機能が必要な場合）
- CORS_ORIGINS: string[]（カンマ区切りのURLリスト）

要件：
- 起動時に検証（エラーは明確なメッセージで表示）
- 欠けている必須変数は全て一覧表示してクラッシュ
- TypeScriptの型を自動エクスポート

保存先: src/config/env.ts
```

---

## 生成される型安全なenv設定

```typescript
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url('DATABASE_URL must be a valid URL'),
  REDIS_URL: z.string().url('REDIS_URL must be a valid URL'),
  JWT_SECRET: z
    .string()
    .min(32, 'JWT_SECRET must be at least 32 characters'),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  NODE_ENV: z
    .enum(['development', 'test', 'production'])
    .default('development'),
  LOG_LEVEL: z
    .enum(['debug', 'info', 'warn', 'error'])
    .default('info'),
  SENDGRID_API_KEY: z.string().optional(),
  CORS_ORIGINS: z
    .string()
    .transform((val) => val.split(',').map((s) => s.trim()))
    .default('http://localhost:3000'),
});

const result = envSchema.safeParse(process.env);

if (!result.success) {
  const errors = result.error.issues.map(
    (issue) => `  - ${issue.path.join('.')}: ${issue.message}`
  );
  console.error('❌ Invalid environment variables:\n' + errors.join('\n'));
  process.exit(1);
}

export const env = result.data;
export type Env = typeof env;
```

---

## .env.exampleの生成

```
src/config/env.tsのスキーマを読んで、.env.exampleを生成してください。

要件：
- 全ての環境変数を列挙
- コメントで説明を追加
- 値は例示値（本物のシークレットは使わない）
- デフォルト値がある場合は表示

保存先: .env.example
```

```bash
# 生成される .env.example
# Database
# PostgreSQL connection string
DATABASE_URL=postgresql://user:password@localhost:5432/myapp

# Redis
# Redis connection URL
REDIS_URL=redis://localhost:6379

# Security
# JWT signing secret (minimum 32 characters, generate with: openssl rand -base64 32)
JWT_SECRET=your-secret-here-minimum-32-characters

# Server
PORT=3000
NODE_ENV=development

# Logging
# Options: debug, info, warn, error
LOG_LEVEL=info

# Email (optional)
# SENDGRID_API_KEY=your-sendgrid-api-key

# CORS
# Comma-separated list of allowed origins
CORS_ORIGINS=http://localhost:3000,http://localhost:3001
```

---

## 環境変数の変更を検出するHook

```python
# .claude/hooks/check_env_usage.py
import json, re, sys

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
fp = data.get("tool_input", {}).get("file_path", "")

if not fp or any(x in fp for x in ["env.ts", "test", "spec"]):
    sys.exit(0)

if not fp.endswith((".ts", ".js")):
    sys.exit(0)

# process.env直接アクセスを検出
if re.search(r'process\.env\.\w+', content):
    print("[ENV] process.env を直接使用しています。", file=sys.stderr)
    print("[ENV] src/config/env.ts 経由でアクセスしてください。", file=sys.stderr)
    sys.exit(2)  # ブロック

sys.exit(0)
```

---

## まとめ

Claude Codeで環境変数管理を設計する：

1. **CLAUDE.md** にenv経由アクセス・検証必須・シークレット管理ルールを明記
2. **Zodスキーマ** で全環境変数を型安全に定義
3. **起動時検証** で設定ミスを即座に検出
4. **.env.example** を常に最新に保つ
5. **Hooks** で `process.env` の直接アクセスをブロック

---

*環境変数の設定ミスとシークレット漏洩リスクを自動検出するスキルは **Security Pack（¥1,480）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
