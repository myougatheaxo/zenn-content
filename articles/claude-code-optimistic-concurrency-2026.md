---
title: "Claude Codeで楽観的並行制御を設計する：バージョニング・競合検出・自動リトライ"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 18:00"
---

## はじめに

「2ユーザーが同時に同じデータを更新したら片方の変更が消えた」——楽観的並行制御でバージョン番号を使って競合を検出し、自動リトライまたはユーザーへの通知でデータ損失を防ぐ設計をClaude Codeに生成させる。

---

## CLAUDE.mdに楽観的並行制御設計ルールを書く

```markdown
## 楽観的並行制御設計ルール

### バージョン管理
- 全てのエンティティにversionフィールド（整数）を持つ
- UPDATE時にWHEREにversion条件を追加（WHERE id=? AND version=?）
- 更新成功でversionをインクリメント（version + 1）
- 更新失敗（0行更新）= 競合発生

### 競合処理
- RETRY可能な操作: 最新データを再取得して自動リトライ（最大3回）
- RETRY不可能な操作: 競合エラーをUIに返してユーザーに再入力を促す
- 楽観的ロック失敗のエラーはHTTP 409 Conflict

### 対象選定
- 競合が稀なリソースに使用（同時編集が多い場合は悲観的ロックを検討）
- 読み取りが多く書き込みが少ないデータが最適
```

---

## 楽観的並行制御実装の生成

```
楽観的並行制御を設計してください。

要件：
- バージョンフィールドによる競合検出
- 自動リトライデコレーター
- 競合時のマージ戦略
- HTTPレスポンスのバージョン管理

生成ファイル: src/concurrency/optimistic/
```

---

## 生成される楽観的並行制御実装

```typescript
// src/concurrency/optimistic/optimisticLock.ts — 楽観的ロック

export class OptimisticLockError extends Error {
  constructor(
    public readonly entityType: string,
    public readonly entityId: string,
    public readonly expectedVersion: number,
    public readonly actualVersion: number
  ) {
    super(`Optimistic lock conflict: ${entityType}[${entityId}] expected v${expectedVersion}, but current is v${actualVersion}`);
  }
}

export class OptimisticLockRepository<T extends { id: string; version: number }> {
  constructor(
    private readonly tableName: string,
    private readonly db: PrismaClient
  ) {}

  async update(
    id: string,
    expectedVersion: number,
    updateData: Partial<Omit<T, 'id' | 'version'>>
  ): Promise<T> {
    // version一致チェック付きでUPDATE
    const result = await this.db.$executeRaw`
      UPDATE ${this.db.sql.raw(this.tableName)}
      SET ${this.buildSetClause(updateData)},
          version = version + 1,
          updated_at = NOW()
      WHERE id = ${id}
        AND version = ${expectedVersion}
      RETURNING *
    `;

    if (result === 0) {
      // 更新行数が0 = バージョン不一致（競合）
      const current = await this.db.$queryRaw<Array<{ version: number }>>`
        SELECT version FROM ${this.db.sql.raw(this.tableName)} WHERE id = ${id}
      `.then(rows => rows[0]);

      throw new OptimisticLockError(
        this.tableName,
        id,
        expectedVersion,
        current?.version ?? -1
      );
    }

    return this.findById(id);
  }

  private buildSetClause(data: Record<string, unknown>): string {
    return Object.entries(data)
      .map(([key, value]) => {
        const snakeKey = key.replace(/[A-Z]/g, c => `_${c.toLowerCase()}`);
        return `${snakeKey} = ${typeof value === 'string' ? `'${value}'` : value}`;
      })
      .join(', ');
  }

  async findById(id: string): Promise<T> {
    const rows = await this.db.$queryRaw<T[]>`
      SELECT * FROM ${this.db.sql.raw(this.tableName)} WHERE id = ${id}
    `;
    if (!rows[0]) throw new NotFoundError(`${this.tableName}[${id}] not found`);
    return rows[0];
  }
}

// 自動リトライデコレーター
export function withOptimisticRetry<T>(
  operation: () => Promise<T>,
  maxAttempts = 3,
  delayMs = 100
): Promise<T> {
  return retry(operation, {
    retries: maxAttempts - 1,
    factor: 2,
    minTimeout: delayMs,
    onRetry: (error, attempt) => {
      if (error instanceof OptimisticLockError) {
        logger.debug({ attempt, entityId: error.entityId }, 'Retrying after optimistic lock conflict');
        return;
      }
      throw error; // OptimisticLockError以外はリトライしない
    },
  });
}
```

```typescript
// src/concurrency/optimistic/userProfileService.ts — 楽観的ロック付きサービス

export class UserProfileService {
  private readonly repo = new OptimisticLockRepository<User>('users', prisma);

  // 自動リトライ: プロフィール更新は競合しても自動解決
  async updateProfile(userId: string, version: number, updates: UpdateProfileInput): Promise<User> {
    return withOptimisticRetry(async () => {
      // 最新バージョンを取得してリトライ
      const current = await this.repo.findById(userId);

      return this.repo.update(userId, current.version, {
        displayName: updates.displayName ?? current.displayName,
        bio: updates.bio ?? current.bio,
        avatarUrl: updates.avatarUrl ?? current.avatarUrl,
      });
    });
  }

  // 競合をユーザーに通知: 残高操作は自動リトライ不可
  async transferBalance(fromId: string, version: number, amount: number): Promise<User> {
    const from = await this.repo.findById(fromId);

    if (from.balance < amount) {
      throw new InsufficientBalanceError();
    }

    try {
      return await this.repo.update(fromId, version, {
        balance: from.balance - amount,
      });
    } catch (error) {
      if (error instanceof OptimisticLockError) {
        // 残高操作は競合を自動解決しない（二重送金リスク）
        throw new ConflictError(
          'Another operation modified this account simultaneously. Please refresh and try again.',
          { currentVersion: error.actualVersion }
        );
      }
      throw error;
    }
  }
}

// REST APIエンドポイント: ETagでバージョン管理
router.get('/api/users/:id', async (req, res) => {
  const user = await userService.findById(req.params.id);

  // ETagにバージョンを埋め込む
  res.set('ETag', `"${user.version}"`);
  res.json(user);
});

router.put('/api/users/:id', async (req, res) => {
  // クライアントはIf-Matchヘッダーでバージョンを指定
  const ifMatch = req.headers['if-match'];
  if (!ifMatch) {
    return res.status(428).json({ error: 'If-Match header required' }); // 428 Precondition Required
  }

  const expectedVersion = parseInt(ifMatch.replace(/"/g, ''));

  try {
    const updated = await userService.updateProfile(req.params.id, expectedVersion, req.body);
    res.set('ETag', `"${updated.version}"`);
    res.json(updated);
  } catch (error) {
    if (error instanceof ConflictError) {
      res.status(409).json({
        error: 'Version conflict',
        message: error.message,
        currentVersion: (error as any).metadata.currentVersion,
      });
    } else {
      throw error;
    }
  }
});

/*
クライアントフロー:
1. GET /api/users/123  → ETag: "5"
2. PUT /api/users/123  If-Match: "5"  → 200 ETag: "6"
3. 別ユーザーが先に PUT → バージョンが"6"に
4. 元ユーザーが PUT /api/users/123 If-Match: "5" → 409 Conflict
5. クライアントが再度 GET → 最新データを取得して再編集
*/
```

---

## まとめ

Claude Codeで楽観的並行制御を設計する：

1. **CLAUDE.md** に全エンティティにversionフィールド・UPDATE時にWHERE version=?を付与・0行更新で競合検出・自動リトライは競合が安全な操作のみを明記
2. **`WHERE id=? AND version=?`** で競合を原子的に検出——チェックしてから更新する間に別トランザクションが割り込む問題がない。0行更新=競合という単純で確実な検出方法
3. **操作別の競合戦略** でリトライの安全性を判断——プロフィール更新は自動リトライOK（最終状態が正しければよい）、残高移動はリトライ禁止（二重送金リスク）
4. **ETag/If-Matchによるバージョン伝播** でHTTP標準仕様でバージョン管理——クライアントがGETで取得したETagをPUTのIf-Matchに使う。RFC 7232準拠のキャッシュ/競合制御

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
