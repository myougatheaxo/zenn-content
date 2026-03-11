---
title: "Claude Codeを使ったレガシーコードのモダナイゼーション実践"
emoji: "🤖"
type: "tech"
topics:
  - claudecode
  - リファクタリング
  - レガシーコード
  - 移行
  - 開発効率
published: true
---

## レガシーコードをどう扱うか

「動いているコードは触るな」というアドバイスがある一方で、レガシーコードを放置するとビジネスリスクになる。

Claude Codeを使ったモダナイゼーションのポイント:
- **一気にやらない** (リスクが高すぎる)
- **まずテストを書く** (変更の安全網を作る)
- **段階的に置き換える** (Strangler Figパターン)

---

## ステップ1: レガシーコードの理解

```bash
claude "src/legacy/ 以下のコードを分析して:
1. このモジュールが何をするか (外部から見た振る舞い)
2. 内部の依存関係マップ
3. 外部から呼び出されているエントリポイント
4. データの入出力形式

テストや変更のために最低限理解が必要な情報に絞って"
```

---

## ステップ2: 既存振る舞いのテスト化

```bash
claude "src/legacy/payment-processor.js の現在の振る舞いを
キャラクタリゼーションテストとして書いて。

目的: リファクタ前の動作を保証する安全網を作ること。
入出力のパターンを網羅的にテスト。
テストが通ることで『変更前と同じ動作』が証明できる状態にする。

出力先: test/legacy/payment-processor.test.js"
```

**キャラクタリゼーションテスト**: コードが「こうあるべき」ではなく「今こう動いている」をテストする手法。

---

## ステップ3: Strangler Figパターンで置き換え

```bash
# まず新しいインターフェースを定義
claude "src/legacy/payment-processor.js の機能を
TypeScriptで再実装するための:
1. インターフェース定義 (src/interfaces/payment-processor.ts)
2. 空の新実装 (src/services/payment-processor-v2.ts)
3. 切り替え用フラグ (config/feature-flags.ts)

今回は骨格だけ作成。実装は次のステップで"
```

---

## ステップ4: 機能単位で移行

```bash
# 1機能ずつ移行
claude "PaymentProcessor の processPayment メソッドを v2 に実装。
- 入力/出力は v1 と完全互換
- 内部実装: async/await, TypeScript, Prisma使用
- キャラクタリゼーションテストが全て通ること
- feature flag 'use_v2_payment' が true の時だけ v2 を使う"
```

---

## ステップ5: データ移行

```bash
claude "legacy DBスキーマから新スキーマへのデータ移行スクリプトを生成:

旧スキーマ (src/legacy/models/):
- users テーブル (id, email, password_hash, created)
- orders テーブル (id, user_id, total, status, created)

新スキーマ (prisma/schema.prisma):
- User (id, email, hashedPassword, createdAt)
- Order (id, userId, totalAmount, status, createdAt, updatedAt)

要件:
- べき等性 (何度実行しても同じ結果)
- ロールバック手順付き
- バッチサイズ 1000件
- 進捗ログ出力"
```

---

## よくある問題と対処

### コールバック地獄 → async/await

```bash
claude "src/legacy/user-service.js のコールバック形式を
全てPromise / async/await に変換。
エラーハンドリングは try/catch に統一。
外部のインターフェースは変えない"
```

### グローバル状態 → 依存性注入

```bash
claude "src/legacy/db.js のグローバルdb接続を
依存性注入パターンに変更。
既存のテストが通ることを確認してから変更すること"
```

### 混在したビジネスロジック → レイヤー分離

```bash
claude "src/legacy/api.js にRouting/Business Logic/DB accessが
混在している。以下のレイヤーに分離:
- routes/ (HTTPのみ)
- services/ (ビジネスロジック)
- repositories/ (DBアクセス)
一度に1ファイルずつ進める"
```

---

## 進捗の可視化

```bash
claude "モダナイゼーションの進捗レポートを生成:
- レガシーコードの残量 (行数・ファイル数)
- 移行完了率
- テストカバレッジの変化
- 残タスクの優先順位

src/legacy/ と src/ を比較して出力"
```

---

## まとめ

| フェーズ | やること | Claude Codeの活用 |
|---------|---------|----------------|
| 理解 | コード分析・依存関係の可視化 | 分析レポート生成 |
| 安全網 | キャラクタリゼーションテスト作成 | テスト自動生成 |
| 置き換え | Strangler Fig + Feature Flag | コード変換 |
| 検証 | テスト通過・パフォーマンス確認 | 比較レポート |

レガシーコードのモダナイゼーションは「テストなしには動かない」。Claude Codeを使っても、この鉄則は変わらない。


---

この記事の内容はnoteで公開しているClaude Code完全攻略ガイド（全7章）から抜粋したもの。MCPの構築方法、カスタムスキル設計、マルチエージェント構成まで網羅している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
