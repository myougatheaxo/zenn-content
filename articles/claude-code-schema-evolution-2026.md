---
title: "Claude CodeでスキーマエボリューションをAPIとDBに適用する：後方互換変更・展開戦略・マイグレーション順序"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-20 20:00"
---

## はじめに

「APIの変更でクライアントが壊れた」「DBカラム追加のデプロイ順序を間違えてサービス停止した」——後方互換を保ちながらAPIとDBスキーマを安全に進化させる設計をClaude Codeに生成させる。

---

## CLAUDE.mdにスキーマエボリューション設計ルールを書く

```markdown
## スキーマエボリューション設計ルール

### APIスキーマの変更ルール（後方互換）
- 追加OK: 新しいフィールド（クライアントはunknownフィールドを無視）
- 追加OK: 新しいオプションパラメーター
- 削除NG: 既存フィールドを即削除しない（deprecatedにして次メジャーバージョンで削除）
- 変更NG: フィールドの型変更、必須→オプション以外の変更

### DBスキーマの変更ルール（ゼロダウンタイム）
- カラム追加: nullable、またはDEFAULT付きで追加
- カラム削除: 3ステップ（アプリが使用停止→カラム削除→コード削除）
- カラム名変更: 4ステップ（新カラム追加→デュアルライト→データコピー→旧カラム削除）
- インデックス追加: CREATE INDEX CONCURRENTLYを使用

### 展開順序
- DB変更を先にデプロイ（後方互換マイグレーション）
- アプリを後にデプロイ
- 古いアプリが新DBと、新アプリが旧DBと共存できること
```

---

## スキーマエボリューション実装の生成

```
スキーマエボリューション戦略を設計してください。

要件：
- APIバージョニングなしの後方互換変更
- DBカラム削除の3ステップ安全削除
- カラム名変更の4ステップ移行
- スキーマ変更チェックリスト

生成ファイル: migrations/ + src/
```

---

## 生成されるスキーマエボリューション実装

```typescript
// ===== APIスキーマエボリューション =====

// ❌ 破壊的変更（やってはいけない）
// Before: { userId: string, email: string }
// After:  { userId: string }  ← emailを削除 → クライアントが壊れる

// ✅ 後方互換変更（safe evolution）

// Step 1: 新フィールドをオプションで追加
// Before
interface UserResponse_v1 {
  userId: string;
  email: string;
  name: string;
}

// After（後方互換）
interface UserResponse_v2 {
  userId: string;
  email: string;
  name: string;
  // 新フィールド: オプション（旧クライアントは無視）
  displayName?: string;   // 追加OK
  avatarUrl?: string;     // 追加OK
  // @deprecated: 次バージョンで削除予定
  legacyUserId?: string;  // 削除前にdeprecatedアノテーション
}

// Step 2: Zodスキーマでの後方互換保証
const UserResponseSchema_v2 = z.object({
  userId: z.string(),
  email: z.string().email(),
  name: z.string(),
  displayName: z.string().optional(),
  avatarUrl: z.string().url().optional(),
  // deprecatedフィールドは残す（undefinedを返す）
  legacyUserId: z.string().optional(),
});

// レスポンス変換（旧クライアントに互換データを返す）
function toUserResponse(user: User): UserResponse_v2 {
  return {
    userId: user.id,
    email: user.email,
    name: user.profile.name,
    displayName: user.profile.displayName,
    avatarUrl: user.profile.avatarUrl ?? undefined,
    // legacyUserIdは段階的に廃止（nullではなくundefinedで省略）
    // legacyUserId: undefined,  ← 最終的にここを削除
  };
}
```

```typescript
// ===== DBスキーマエボリューション =====

// ===== カラム追加（安全） =====
// migration: add_display_name_to_users.sql
/*
-- Nullableカラム追加（既存データに影響なし）
ALTER TABLE users ADD COLUMN display_name TEXT;

-- NOT NULLにしたい場合はDEFAULTを付けてから変更
ALTER TABLE users ADD COLUMN is_verified BOOLEAN NOT NULL DEFAULT FALSE;
*/

// ===== カラム削除（3ステップ安全削除） =====

// Step 1: アプリがカラムを使用停止（コードを先にデプロイ）
// src/infrastructure/repositories/userRepository.ts
class PrismaUserRepository {
  async findById(id: string): Promise<User | null> {
    const row = await this.db.user.findUnique({
      where: { id },
      select: {
        id: true,
        email: true,
        name: true,
        // legacy_user_id: true,  ← この行を削除（Step 1）
      },
    });
    return row ? UserMapper.toDomain(row) : null;
  }
}

// Step 2: DBから論理削除（カラム名を変更してアプリが使えなくする）
// migration: step2_soft_delete_legacy_user_id.sql
/*
-- アプリがカラムを使っていないことを確認してから実行
ALTER TABLE users RENAME COLUMN legacy_user_id TO _deprecated_legacy_user_id;
*/

// Step 3: カラムを物理削除
// migration: step3_drop_legacy_user_id.sql
/*
-- 問題がないことを1週間確認後に実行
ALTER TABLE users DROP COLUMN _deprecated_legacy_user_id;
*/
```

```typescript
// ===== カラム名変更（4ステップ移行） =====

// ユーザー名: name → full_name への変更例

// Step 1: 新カラムを追加
// migration: step1_add_full_name.sql
/*
ALTER TABLE users ADD COLUMN full_name TEXT;
*/

// Step 2: デュアルライト（新旧両方に書き込む）
// src/infrastructure/repositories/userRepository.ts
class PrismaUserRepository {
  async save(user: User): Promise<void> {
    await this.db.$executeRaw`
      INSERT INTO users (id, email, name, full_name)
      VALUES (${user.id}, ${user.email}, ${user.profile.name}, ${user.profile.name})
      ON CONFLICT (id) DO UPDATE SET
        email = EXCLUDED.email,
        name = EXCLUDED.name,       -- 旧カラムにも書き込む（後方互換）
        full_name = EXCLUDED.full_name  -- 新カラムにも書き込む
    `;
  }
}

// Step 3: データの一括コピー + 読み取りを新カラムに移行
// migration: step3_backfill_full_name.sql
/*
-- バッチでコピー（ロックなし）
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- NOT NULLを付与
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;
*/

// コードを新カラムから読み取るように変更
async findById(id: string): Promise<User | null> {
  const row = await this.db.user.findUnique({ where: { id } });
  return row ? {
    ...row,
    name: row.fullName,  // 新カラムから読み取り
  } : null;
}

// Step 4: 旧カラム削除（Step 1のデュアルライトを停止後）
// migration: step4_drop_name.sql
/*
ALTER TABLE users DROP COLUMN name;
*/

// ===== インデックス追加（ダウンタイムなし） =====
// migration: add_index_concurrently.sql
/*
-- CONCURRENTLY: テーブルをロックせずにインデックスを作成
-- Prismaのmigrateでは自動生成されないため手書き必須
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- 失敗した場合はINVALIDインデックスが残る → DROP後に再実行
-- DROP INDEX CONCURRENTLY IF EXISTS idx_users_email;
*/
```

```typescript
// スキーマ変更チェックリスト自動生成
// scripts/schema-change-checker.ts

const BREAKING_CHANGES_PATTERNS = [
  /ALTER TABLE .+ DROP COLUMN/i,     // カラム削除
  /ALTER TABLE .+ RENAME COLUMN/i,   // カラム名変更
  /ALTER TABLE .+ ALTER COLUMN .+ SET NOT NULL/i,  // NOTNULLへの変更
  /DROP TABLE/i,                     // テーブル削除
  /CREATE INDEX(?! CONCURRENTLY)/i,  // CONCURRENTLYなしのインデックス作成
];

function checkMigration(sql: string): string[] {
  const warnings: string[] = [];
  for (const pattern of BREAKING_CHANGES_PATTERNS) {
    if (pattern.test(sql)) {
      warnings.push(`⚠️ Potentially breaking change detected: ${pattern.source}`);
    }
  }
  return warnings;
}

// CI: マイグレーションファイルに破壊的変更が含まれていないかチェック
const migrationSql = readFileSync('migrations/new_migration.sql', 'utf8');
const warnings = checkMigration(migrationSql);
if (warnings.length > 0) {
  console.warn('Migration safety warnings:\n', warnings.join('\n'));
  // CIは警告のみ（レビュー必須にする）
}
```

---

## まとめ

Claude Codeでスキーマエボリューションを設計する：

1. **CLAUDE.md** にAPIフィールド追加はOK・削除前にdeprecated宣言必須・DB変更は先にデプロイ・アプリ変更は後にデプロイ・`CREATE INDEX CONCURRENTLY`を明記
2. **カラム削除の3ステップ** でゼロダウンタイム——①アプリがカラム参照を停止②DBでリネーム（`_deprecated_`プレフィックス）③1週間後に物理削除。古いアプリが新DBで動ける期間を確保
3. **カラム名変更の4ステップ** ——①新カラム追加②デュアルライト（旧→新）③バックフィル+読み取り切替④旧カラム削除。デュアルライト期間中はどのバージョンのアプリも動作する
4. **`CREATE INDEX CONCURRENTLY`** でテーブルロックなしにインデックス追加——通常の`CREATE INDEX`は書き込みをブロックする。`CONCURRENTLY`は遅いがロックなし。Prismaが自動生成しないため手書きマイグレーションファイルが必要

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
