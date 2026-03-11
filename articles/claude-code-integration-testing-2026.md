---
title: "Claude Codeで統合テストを設計する：APIテストとDBテストの自動化パターン"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "testing", "nodejs", "typescript", "テスト"]
published: true
---

## はじめに：単体テストだけでは足りない理由

単体テストを完璧に書いても、モジュール間の連携バグは見つけられない。

よくある失敗パターン：
- `UserService.createUser()` の単体テストは全てパス
- `POST /api/users` のAPIテストも単体では正常
- しかし実際にリクエストを送ると、バリデーションエラーが400ではなく500で返る
- 原因：ミドルウェアのエラーハンドラーとサービス層の例外型が噛み合っていなかった

単体テストはモックで隔離するため、こういった「つなぎ目のバグ」を見逃す。統合テストは実際のHTTPリクエスト・実DBを通して、システム全体の動作を検証する。

Claude Codeに統合テストを生成させる際も、「何をテストするか」をCLAUDE.mdで明示しないと、モックだらけの名ばかり統合テストが生成される。

---

## CLAUDE.mdに統合テストルールを書く

プロジェクトルートの`CLAUDE.md`に以下を追加する：

```markdown
## 統合テスト規約

### テストフレームワーク
- HTTPテスト: supertest + vitest
- DBアクセス: Prisma（実DBを使う、モック禁止）
- テスト用DB: Dockerで起動（`docker-compose.test.yml`）

### テスト分類
- 単体テスト: `*.unit.test.ts` — モック使用可
- 統合テスト: `*.integration.test.ts` — 実DB・実HTTP必須

### 統合テストのルール
1. `beforeAll`でDB接続、`afterAll`でDB切断
2. 各テストは`beforeEach`でDBをクリーンな状態にリセット
3. テストデータはシーディング関数で挿入（インラインのSQLやハードコード禁止）
4. HTTPテストはsupertestの`request(app)`を使う（サーバーを実際に起動しない）
5. レスポンスのステータスコード・ボディ・DB状態を全て検証する

### テスト用DB設定
- 環境変数: `DATABASE_URL_TEST`
- 起動: `docker-compose -f docker-compose.test.yml up -d`
- マイグレーション: `DATABASE_URL=$DATABASE_URL_TEST npx prisma migrate deploy`
```

このルールをCLAUDE.mdに書いておくと、Claude Codeが統合テストを生成する際に自動的に参照する。

---

## APIエンドポイントの統合テスト生成プロンプト

### プロンプト例

```
CLAUDE.mdの統合テスト規約に従い、以下のAPIエンドポイントの統合テストを生成してください。

対象: POST /api/users（ユーザー作成）
仕様:
- バリデーション: email必須・形式チェック、name必須・最大50文字
- 成功時: 201 + 作成されたユーザーオブジェクト（passwordHashは除外）
- バリデーションエラー: 400 + エラー詳細
- メールアドレス重複: 409 + エラーメッセージ

使用技術: supertest, vitest, Prisma
ファイル: src/__tests__/users.integration.test.ts
```

### 生成される統合テストの例

```typescript
// src/__tests__/users.integration.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import request from "supertest";
import { app } from "../../app";
import { prisma } from "../../lib/prisma";

beforeAll(async () => {
  await prisma.$connect();
});

afterAll(async () => {
  await prisma.$disconnect();
});

beforeEach(async () => {
  // テスト間の独立性確保：全テーブルをクリア（外部キー順に削除）
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
});

describe("POST /api/users", () => {
  it("正常なユーザーを作成できる", async () => {
    const res = await request(app)
      .post("/api/users")
      .send({ name: "みょうが", email: "myouga@example.com" })
      .expect(201);

    expect(res.body).toMatchObject({
      id: expect.any(String),
      name: "みょうが",
      email: "myouga@example.com",
    });
    // passwordHashが返らないことを確認
    expect(res.body.passwordHash).toBeUndefined();

    // DBに実際に保存されているか確認
    const user = await prisma.user.findUnique({
      where: { email: "myouga@example.com" },
    });
    expect(user).not.toBeNull();
    expect(user!.name).toBe("みょうが");
  });

  it("メールアドレスが無効な場合は400を返す", async () => {
    const res = await request(app)
      .post("/api/users")
      .send({ name: "みょうが", email: "invalid-email" })
      .expect(400);

    expect(res.body.error).toContain("email");
  });

  it("メールアドレスが重複している場合は409を返す", async () => {
    // 先にユーザーを作成
    await prisma.user.create({
      data: { name: "先住民", email: "myouga@example.com" },
    });

    const res = await request(app)
      .post("/api/users")
      .send({ name: "みょうが", email: "myouga@example.com" })
      .expect(409);

    expect(res.body.error).toContain("already exists");
  });

  it("nameが50文字を超える場合は400を返す", async () => {
    await request(app)
      .post("/api/users")
      .send({ name: "あ".repeat(51), email: "myouga@example.com" })
      .expect(400);
  });
});
```

supertestは`app`オブジェクトを受け取ってHTTPリクエストを送る。サーバーを実際に起動しないため、ポート競合が起きない。

---

## テストデータのシーディング生成プロンプト

テスト間でデータを共有すると、テストの実行順序に依存したフラキーテストが生まれる。シーディング関数でテストデータを明示的に管理する。

### プロンプト例

```
CLAUDE.mdの統合テスト規約に従い、以下のテストシーディング関数を生成してください。

要件:
- ファイル: src/__tests__/helpers/seed.ts
- 各テストから呼び出せるシーディング関数を定義
- Prismaを使ってDBにテストデータを挿入
- 戻り値は挿入したレコードのオブジェクト（IDを含む）

必要なシーディング関数:
1. seedUser(overrides?) — ユーザーを1件作成
2. seedUserWithPosts(postCount) — ユーザーと指定数の投稿を作成
3. seedAdminUser() — 管理者ユーザーを作成
```

### 生成されるシーディング関数の例

```typescript
// src/__tests__/helpers/seed.ts
import { prisma } from "../../lib/prisma";
import type { User, Post } from "@prisma/client";

let userCounter = 0;

// ユーザーを1件作成（メールアドレスはユニークになるよう連番を使用）
export async function seedUser(
  overrides: Partial<{ name: string; email: string; role: string }> = {}
): Promise<User> {
  userCounter++;
  return prisma.user.create({
    data: {
      name: overrides.name ?? `テストユーザー${userCounter}`,
      email: overrides.email ?? `test${userCounter}@example.com`,
      role: overrides.role ?? "user",
    },
  });
}

// ユーザーと投稿をまとめて作成
export async function seedUserWithPosts(
  postCount: number
): Promise<{ user: User; posts: Post[] }> {
  const user = await seedUser();
  const posts = await Promise.all(
    Array.from({ length: postCount }, (_, i) =>
      prisma.post.create({
        data: {
          title: `テスト投稿${i + 1}`,
          content: `本文${i + 1}`,
          authorId: user.id,
          published: true,
        },
      })
    )
  );
  return { user, posts };
}

// 管理者ユーザーを作成
export async function seedAdminUser(): Promise<User> {
  return seedUser({ name: "管理者", email: "admin@example.com", role: "admin" });
}
```

### シーディング関数を使ったテスト例

```typescript
import { seedUser, seedUserWithPosts } from "./helpers/seed";

describe("GET /api/users/:id", () => {
  it("存在するユーザーを取得できる", async () => {
    // シーディング関数でテストデータを作成（IDが確定する）
    const user = await seedUser({ name: "みょうが" });

    const res = await request(app)
      .get(`/api/users/${user.id}`)
      .expect(200);

    expect(res.body.name).toBe("みょうが");
  });

  it("投稿数を含むユーザー情報を取得できる", async () => {
    const { user } = await seedUserWithPosts(3);

    const res = await request(app)
      .get(`/api/users/${user.id}?include=posts`)
      .expect(200);

    expect(res.body.posts).toHaveLength(3);
  });
});
```

`beforeEach`でDBをクリアし、各テストでシーディング関数を呼ぶことで、テスト間の独立性が完全に保たれる。

---

## データベース統合テストプロンプト

トランザクションやロールバックを使ったDB操作は、単体テストでは正確に検証できない。Prismaのインタラクティブトランザクションを使った統合テストを生成させる。

### プロンプト例

```
CLAUDE.mdの統合テスト規約に従い、以下のサービス層の統合テストを生成してください。

対象: UserService.transferPosts(fromUserId, toUserId)
仕様:
- fromUserの全投稿をtoUserに移管する
- fromUserが存在しない場合はNotFoundErrorをスローしてロールバック
- toUserが存在しない場合はNotFoundErrorをスローしてロールバック
- 移管は全件成功か全件失敗（トランザクション）

テスト観点:
1. 正常移管: DBの状態を直接確認する
2. fromUser不在: エラー後にDBが変化していないことを確認
3. 移管後のfromUserの投稿数が0になることを確認
```

### 生成されるDBトランザクションテストの例

```typescript
// src/__tests__/user-service.integration.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { UserService } from "../../services/UserService";
import { prisma } from "../../lib/prisma";
import { NotFoundError } from "../../errors";
import { seedUser, seedUserWithPosts } from "./helpers/seed";

const userService = new UserService(prisma);

beforeAll(async () => { await prisma.$connect(); });
afterAll(async () => { await prisma.$disconnect(); });
beforeEach(async () => {
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
});

describe("UserService.transferPosts", () => {
  it("全投稿が正しく移管される", async () => {
    const { user: from, posts } = await seedUserWithPosts(3);
    const to = await seedUser();

    await userService.transferPosts(from.id, to.id);

    // fromUserの投稿が0になっていることをDB直接確認
    const fromPostCount = await prisma.post.count({
      where: { authorId: from.id },
    });
    expect(fromPostCount).toBe(0);

    // toUserが全投稿を持っていることを確認
    const toPostCount = await prisma.post.count({
      where: { authorId: to.id },
    });
    expect(toPostCount).toBe(3);
  });

  it("fromUserが存在しない場合はロールバックされる", async () => {
    const to = await seedUser();
    const nonExistentId = "00000000-0000-0000-0000-000000000000";

    await expect(
      userService.transferPosts(nonExistentId, to.id)
    ).rejects.toThrow(NotFoundError);

    // toUserの投稿が0のままであることを確認（ロールバック検証）
    const toPostCount = await prisma.post.count({
      where: { authorId: to.id },
    });
    expect(toPostCount).toBe(0);
  });

  it("toUserが存在しない場合は移管されない", async () => {
    const { user: from } = await seedUserWithPosts(2);
    const nonExistentId = "00000000-0000-0000-0000-000000000000";

    await expect(
      userService.transferPosts(from.id, nonExistentId)
    ).rejects.toThrow(NotFoundError);

    // fromUserの投稿が変化していないことを確認
    const fromPostCount = await prisma.post.count({
      where: { authorId: from.id },
    });
    expect(fromPostCount).toBe(2);
  });
});
```

ポイントは「エラー後にDBの状態を確認する」こと。ロールバックが正しく機能しているかを、DBを直接クエリして検証する。

---

## まとめ

Claude Codeに統合テストを生成させるための3つの原則：

| 原則 | 実践 |
|------|------|
| CLAUDE.mdでルールを明示 | 実DB使用・モック禁止・クリーンアップ方法を記述 |
| シーディング関数で独立性確保 | `beforeEach`でリセット + 各テストでデータ作成 |
| DBの状態も検証する | HTTPレスポンスだけでなく、Prismaで直接クエリ確認 |

この3原則をCLAUDE.mdに書いておくだけで、Claude Codeが生成する統合テストの品質が大幅に上がる。

---

テストカバレッジの抜け漏れを自動検出したい場合は、Claude Code カスタムスキル **Code Review Pack**（¥980）の `/code-review` コマンドが使えます。テストカバレッジ・境界値・エラーパスの見落としを5軸で自動レポートします。

👉 [https://prompt-works.jp](https://prompt-works.jp)
