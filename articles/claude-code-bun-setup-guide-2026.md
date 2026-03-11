---
title: "Bunランタイム環境構築ガイド：Node.jsから移行する実践パターン"
emoji: "🍞"
type: "tech"
topics:
  - claudecode
  - bun
  - typescript
  - nodejs
published: true
---

## Bunとは何か：Node.jsとの違い

Bunは2023年に正式リリースされた、JavaScriptランタイム・バンドラー・パッケージマネージャーを一体化したツールだ。JavaScriptCoreエンジン（SafariのJS エンジン）をベースにZigで実装されており、Node.jsより2〜3倍高速に動作する。

Bunが速い理由はアーキテクチャにある。起動時間がNode.jsの1/3程度、ネイティブHTTPサーバーのスループットはExpressの約4倍、`bun install` はnpmの約25倍速い。

ただし2026年時点での注意点:
- Node.js完全互換ではない（一部APIが未実装）
- エコシステムは成熟途上
- プロダクション採用は用途を選ぶ

**Bunが向いているケース**: スクリプト実行・内部ツール・マイクロサービス・TypeScriptプロジェクト

## インストールと初期設定

```bash
# インストール
curl -fsSL https://bun.sh/install | bash

# バージョン確認
bun --version  # 1.1.x

# 新しいプロジェクトを作成
bun init

# TypeScriptプロジェクト
bun init -y
```

`bun init` で生成される `package.json`:

```json
{
  "name": "my-bun-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "bun run --hot src/index.ts",
    "start": "bun run src/index.ts",
    "build": "bun build src/index.ts --outdir dist --target node",
    "test": "bun test"
  },
  "devDependencies": {
    "@types/bun": "^1.1.0"
  }
}
```

`bunfig.toml`（Bun固有の設定ファイル）:

```toml
[install]
# lockfileのフォーマット
lockfile = "bun.lockb"

[install.scopes]
# プライベートレジストリの設定
"@myorg" = { url = "https://registry.myorg.com/", token = "$NPM_TOKEN" }

[test]
# テストのタイムアウト設定
timeout = 10000
coverage = true
```

## Bun HTTPサーバー：高速Webサーバーの実装

```typescript
// src/server.ts
const PORT = parseInt(process.env.PORT ?? "3000");

const server = Bun.serve({
  port: PORT,
  hostname: "0.0.0.0",

  async fetch(req: Request): Promise<Response> {
    const url = new URL(req.url);

    // ルーティング
    if (url.pathname === "/health") {
      return Response.json({ status: "ok", uptime: process.uptime() });
    }

    if (url.pathname.startsWith("/api/")) {
      return handleApi(req, url);
    }

    return new Response("Not Found", { status: 404 });
  },

  error(error: Error): Response {
    console.error("Server error:", error);
    return Response.json({ error: "Internal Server Error" }, { status: 500 });
  },
});

async function handleApi(req: Request, url: URL): Promise<Response> {
  const path = url.pathname.slice(5); // /api/ を除去

  if (path === "users" && req.method === "GET") {
    const users = await db.users.findAll();
    return Response.json(users);
  }

  if (path === "users" && req.method === "POST") {
    const body = await req.json();
    const user = await db.users.create(body);
    return Response.json(user, { status: 201 });
  }

  return Response.json({ error: "Not found" }, { status: 404 });
}

console.log(`Server running at http://localhost:${server.port}`);
```

## Bunのネイティブファイル操作

```typescript
// Bun固有のファイルAPIはNode.jsより簡潔
async function readJsonFile<T>(path: string): Promise<T> {
  const file = Bun.file(path);
  return file.json<T>();
}

async function writeJsonFile(path: string, data: unknown): Promise<void> {
  await Bun.write(path, JSON.stringify(data, null, 2));
}

// テキストファイルの読み書き
const content = await Bun.file("./config.txt").text();
await Bun.write("./output.txt", "Hello, Bun!");

// バイナリデータ
const buffer = await Bun.file("./image.png").arrayBuffer();

// ストリームとして読む
const stream = Bun.file("./large-file.csv").stream();
const reader = stream.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  // process value...
}

// テンポラリファイル
const tmpFile = Bun.file(Bun.tempFile());
await Bun.write(tmpFile, "temporary data");
```

## Bunテストランナー

```typescript
// src/__tests__/users.test.ts
import { describe, it, expect, beforeEach, mock } from "bun:test";

// モジュールモック
const mockDb = {
  users: {
    findAll: mock(() => Promise.resolve([{ id: 1, name: "Test User" }])),
    create: mock((data: unknown) => Promise.resolve({ id: 2, ...data as object })),
  },
};

describe("Users API", () => {
  beforeEach(() => {
    mockDb.users.findAll.mockReset();
    mockDb.users.create.mockReset();
  });

  it("GET /api/users returns user list", async () => {
    mockDb.users.findAll.mockResolvedValue([{ id: 1, name: "Alice" }]);

    const req = new Request("http://localhost/api/users");
    const res = await handleApi(req, new URL(req.url));
    const body = await res.json();

    expect(res.status).toBe(200);
    expect(body).toHaveLength(1);
    expect(body[0].name).toBe("Alice");
  });

  it("POST /api/users creates a user", async () => {
    const userData = { name: "Bob", email: "bob@example.com" };
    mockDb.users.create.mockResolvedValue({ id: 3, ...userData });

    const req = new Request("http://localhost/api/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(userData),
    });
    const res = await handleApi(req, new URL(req.url));

    expect(res.status).toBe(201);
    expect(mockDb.users.create).toHaveBeenCalledWith(userData);
  });
});

// スナップショットテスト
it("generates correct HTML", () => {
  const html = renderComponent({ title: "Test" });
  expect(html).toMatchSnapshot();
});
```

## Node.jsプロジェクトからの移行手順

```bash
# 1. package-lock.json を削除して bun install
rm -f package-lock.json yarn.lock
bun install  # node_modules と bun.lockb を生成

# 2. npm scripts を bun run に置き換え
# package.json の scripts はそのまま動く

# 3. 互換性チェック
bun run check-compatibility.ts
```

```typescript
// check-compatibility.ts
// Node.js固有のAPIがどこで使われているか確認

const NODE_ONLY_APIS = [
  "require(",
  "__dirname",
  "__filename",
  "process.mainModule",
];

const files = await Array.fromAsync(
  new Bun.Glob("src/**/*.{ts,js}").scan(".")
);

for (const file of files) {
  const content = await Bun.file(file).text();
  for (const api of NODE_ONLY_APIS) {
    if (content.includes(api)) {
      console.warn(`⚠️  ${file}: Uses Node-only API: ${api}`);
    }
  }
}
console.log("Compatibility check complete");
```

## BunのSQLiteネイティブ統合

```typescript
// Bunはsqlite3を組み込みでサポート（node-sqliteは不要）
import { Database } from "bun:sqlite";

const db = new Database("myapp.db", { create: true });

// WALモードで高速化
db.run("PRAGMA journal_mode = WAL");
db.run("PRAGMA synchronous = NORMAL");

// テーブル作成
db.run(`
  CREATE TABLE IF NOT EXISTS users (
    id   INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT    NOT NULL,
    email TEXT   UNIQUE NOT NULL,
    created_at TEXT DEFAULT (datetime('now'))
  )
`);

// プリペアドステートメント
const insertUser = db.prepare(
  "INSERT INTO users (name, email) VALUES ($name, $email)"
);
const findUser = db.prepare("SELECT * FROM users WHERE id = $id");

// トランザクション
const createUsers = db.transaction((users: Array<{name: string; email: string}>) => {
  for (const user of users) {
    insertUser.run({ $name: user.name, $email: user.email });
  }
});

createUsers([
  { name: "Alice", email: "alice@example.com" },
  { name: "Bob", email: "bob@example.com" },
]);

const user = findUser.get({ $id: 1 });
console.log(user);
```

Bunは「速さが必要なスクリプト・内部ツール・TypeScriptプロジェクト」に特に威力を発揮する。Node.jsとの完全互換ではないため、既存の大規模プロジェクトへの移行は慎重に進める必要があるが、新規プロジェクトでの採用は十分な選択肢だ。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
