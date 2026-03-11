---
title: "Claude CodeでTurborepoを設計する：共有型定義・ビルドキャッシュ・pnpm workspace"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "turborepo", "monorepo"]
published: true
---

## はじめに

フロントエンド・バックエンドを別リポジトリで管理すると、型定義の同期が地獄になる。Turborepoでモノレポに統合してビルドキャッシュで高速化する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにモノレポルールを書く

```markdown
## Turborepoモノレポ設計ルール

### ディレクトリ構成
- apps/: デプロイ可能なアプリ（api, web, admin）
- packages/: 共有パッケージ（ui, types, config）
- packages/types: APIの型定義（フロント・バックで共有）

### ビルド依存関係
- turborepo.json で依存グラフを定義
- ^build: 上流パッケージを先にビルド
- outputsでキャッシュ対象ディレクトリを指定

### CI最適化
- Turborepoリモートキャッシュ（Vercel or セルフホスト）
- 変更されたパッケージのみビルド（--filter）
- pnpm workspaceで依存関係を管理
```

---

## turbo.json設定

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"],
      "cache": true
    },
    "test": {
      "dependsOn": ["^build"],
      "cache": false
    },
    "lint": { "outputs": [] },
    "dev": { "cache": false, "persistent": true }
  }
}
```

---

## 共有型定義パッケージ

```typescript
// packages/types/src/user.ts
export interface User {
  id: string;
  email: string;
  name: string;
  role: 'user' | 'admin';
  createdAt: string; // ISO 8601
}

export interface UserListResponse {
  data: User[];
  nextCursor: string | null;
  hasNextPage: boolean;
}
```

```json
// packages/types/package.json
{
  "name": "@myapp/types",
  "version": "0.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": { "build": "tsc" }
}
```

```json
// apps/api/package.json
{
  "name": "@myapp/api",
  "dependencies": {
    "@myapp/types": "workspace:*"
  }
}
```

```typescript
// apps/api/src/routes/users.ts
import type { UserListResponse } from '@myapp/types'; // 共有型を使用

router.get('/users', async (req, res) => {
  const result = await listUsers();
  const response: UserListResponse = {
    data: result.users,
    nextCursor: result.nextCursor,
    hasNextPage: result.hasNextPage,
  };
  res.json(response);
});
```

---

## GitHub Actions CI

```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile

      # Turborepoリモートキャッシュ
      - name: Build
        run: pnpm turbo build
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}

      # 変更されたパッケージのみテスト
      - run: pnpm turbo test --filter='[HEAD^1]'
```

---

## まとめ

Claude CodeでTurborepoモノレポを設計する：

1. **CLAUDE.md** にディレクトリ構成・依存グラフ・キャッシュ設定を明記
2. **packages/types** で型定義を共有（フロント・バックが常に同期）
3. **^build** で上流パッケージのビルド完了を待つ
4. **--filter='[HEAD^1]'** で変更のあったパッケージのみCIで実行

---

*モノレポの設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
