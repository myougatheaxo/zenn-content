---
title: "モノレポ環境でClaude Codeを使う：CLAUDE.md階層化とスコープ管理"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "monorepo", "turborepo", "typescript", "開発効率化"]
published: true
---

## はじめに

モノレポ（Monorepo）環境ではClaude Codeの設定が少し複雑になる。複数のパッケージが混在するため、CLAUDE.mdの階層化とスコープ管理が重要だ。

---

## CLAUDE.mdの階層化

モノレポでは `CLAUDE.md` を複数階層に配置できる。

```
my-monorepo/
├── CLAUDE.md              # 全パッケージ共通の設定
├── packages/
│   ├── api/
│   │   └── CLAUDE.md      # API固有の設定
│   ├── web/
│   │   └── CLAUDE.md      # フロントエンド固有の設定
│   └── shared/
│       └── CLAUDE.md      # 共有パッケージの設定
└── apps/
    └── mobile/
        └── CLAUDE.md      # モバイルアプリ固有の設定
```

Claude Codeは作業ディレクトリに最も近い `CLAUDE.md` を優先しつつ、親ディレクトリの `CLAUDE.md` も読み込む。

---

## ルートのCLAUDE.md（全パッケージ共通）

```markdown
# モノレポ: my-monorepo

## 構成
- パッケージマネージャー: pnpm (workspaces)
- ビルドシステム: Turborepo
- 言語: TypeScript 5.x (全パッケージ)

## コマンド
- 全パッケージビルド: `pnpm turbo build`
- 全テスト実行: `pnpm turbo test`
- 特定パッケージ: `pnpm --filter @myapp/api build`
- 依存関係追加: `pnpm --filter @myapp/api add <pkg>`

## 共通ルール（全パッケージ）
- TypeScript strict: true
- any型禁止
- console.log禁止
- 外部パッケージのバージョンはルートの package.json で管理

## パッケージ間の依存関係
- api は shared に依存可（web に依存不可）
- web は shared に依存可（api に直接依存不可）
- shared は他パッケージに依存不可（循環依存防止）

## 変更禁止
- packages/shared/src/types/ の型定義（全パッケージへの影響大）
- pnpm-lock.yaml を手動編集しない
```

---

## パッケージ固有のCLAUDE.md

APIパッケージ固有の設定：

```markdown
# packages/api

## このパッケージの役割
バックエンドAPIサーバー（Express + Prisma）

## 技術スタック
- Express 4 + Prisma ORM
- PostgreSQL 16
- Jest + Supertest

## アーキテクチャ
- Router → Controller → Service → Repository
- ControllerにDB操作を書かない

## パスエイリアス
- `@myapp/shared` = packages/shared/src
- `@api/lib` = src/lib

## DBマイグレーション
`pnpm --filter @myapp/api prisma:migrate`
```

---

## スコープを明示してClaude Codeに指示する

モノレポでは、どのパッケージの作業をしているか明示する。

```
`packages/api` パッケージで作業しています。
shared パッケージの型は変更せず、api パッケージ内だけで完結する実装をしてください。

[作業内容]
```

---

## パッケージ間の型共有

shared パッケージの型定義をClaude Codeが正しく使えるようにする。

```markdown
## packages/shared の型定義
- ユーザー型: packages/shared/src/types/user.ts
- APIレスポンス型: packages/shared/src/types/api.ts
- エラー型: packages/shared/src/types/errors.ts

これらの型を変更する場合は必ずチームに確認が必要。
全パッケージに影響するため。
```

---

## Turborepo設定のCLAUDE.md参照

```markdown
## Turborepoの依存関係グラフ
- build: web は api に依存（api が先にビルドされる）
- test: 各パッケージが独立してテスト実行
- lint: 各パッケージが独立してLint実行
- deploy: api → web の順でデプロイ

Turboの設定ファイル: turbo.json
```

---

## まとめ

モノレポでのClaude Code設定：

1. **CLAUDE.mdを階層化**（共通設定 + パッケージ固有設定）
2. **パッケージ間の依存関係ルール**を明示
3. **作業スコープ**を毎回明示（どのパッケージの作業か）
4. **shared パッケージの型**は変更制約を厳しく設定

---

*モノレポ環境向けのClaude Codeスキルセットは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
