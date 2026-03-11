---
title: "Claude CodeでミューテーションテストをTypeScriptに適用する：Stryker・テスト品質の測定・サバイバー分析"
emoji: "🧬"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "testing", "architecture"]
published: true
published_at: "2026-03-20 17:00"
---

## はじめに

「カバレッジ100%なのにバグが出た」「テストがあるのに仕様変更を検知できなかった」——ミューテーションテストでコードに意図的なバグ（ミュータント）を注入し、テストがそれを検出できるかを検証する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにミューテーションテスト設計ルールを書く

```markdown
## ミューテーションテスト設計ルール

### なぜミューテーションテストか
- カバレッジ: コードが実行されたかを計測（質は測れない）
- ミューテーションスコア: テストがバグを検出できるかを計測（質を測る）
- 目標: ミューテーションスコア80%以上（ドメインロジックは90%+）

### Stryker設定
- ライブラリ: @stryker-mutator/core + @stryker-mutator/typescript-checker
- 対象: src/domain/** (優先), src/application/**
- 除外: src/infrastructure/**, src/adapters/**（インフラのミューテーションは低ROI）

### サバイバー対応
- サバイバー（検出できなかったミュータント）を分析してテストを追加
- 等価ミュータント（バグだが挙動が変わらない）は `// Stryker disable` でスキップ
```

---

## ミューテーションテスト実装の生成

```
Strykerによるミューテーションテストを設計してください。

要件：
- stryker.config.ts の設定
- ドメインロジックへの適用
- サバイバー分析とテスト追加
- CIパイプライン統合

生成ファイル: stryker.config.ts + src/__tests__/
```

---

## 生成されるミューテーションテスト実装

```typescript
// stryker.config.ts — Stryker設定

import type { Config } from '@stryker-mutator/api/config';

const config: Config = {
  packageManager: 'npm',
  reporters: ['html', 'clear-text', 'progress', 'json'],
  testRunner: 'jest',
  coverageAnalysis: 'perTest',

  // ミューテーションの対象（ドメインとアプリケーション層を優先）
  mutate: [
    'src/domain/**/*.ts',
    'src/application/**/*.ts',
    // インフラは除外（外部APIのミューテーションはテストコストが高い）
    '!src/infrastructure/**',
    '!src/**/*.test.ts',
    '!src/**/*.spec.ts',
  ],

  // TypeScriptチェッカー（型エラーのミュータントを除外）
  checkers: ['typescript'],
  tsconfigFile: 'tsconfig.json',

  // ミューテーター設定（不要なミュータントを無効化）
  mutator: {
    plugins: [
      'arithmetic',           // +, -, *, / の置換
      'boolean-literal',      // true/false の反転
      'conditional-expression', // 条件式の変更
      'equality',             // ===, !==, <, > 等の変更
      'logical-operator',     // &&, || の変更
      'string-literal',       // 文字列の変更（空文字化）
      'unary-operator',       // !, ++ 等の変更
      'block-statement',      // ブロック全体を空にする
    ],
  },

  // 閾値設定
  thresholds: {
    high: 90,    // 90%以上: 緑
    low: 80,     // 80%以上: 黄色
    break: 70,   // 70%未満: CIを失敗させる
  },

  // タイムアウト設定（重いミュータントに対応）
  timeoutMS: 60_000,
  concurrency: 4,
};

export default config;
```

```typescript
// ミューテーションテストで発見されたサバイバーへの対応例

// === サバイバー1: 境界値の条件式 ===
// 元コード
class OrderItem {
  static create(quantity: number): OrderItem {
    if (quantity <= 0) throw new DomainError('Quantity must be positive');
    // Strykerが quantity < 0 に変更 → テストが検出できなかった（サバイバー）
    ...
  }
}

// 追加すべきテスト（サバイバーを殺す）
describe('OrderItem.create', () => {
  // 既存テスト: quantity=0 でエラー
  it('quantity=0はエラー', () => {
    expect(() => OrderItem.create(0)).toThrow(DomainError);
  });

  // 追加テスト: quantity=-1 でもエラー（<= の両側を検証）
  it('quantity=-1はエラー', () => {
    expect(() => OrderItem.create(-1)).toThrow(DomainError);
  });

  // 追加テスト: quantity=1 は成功（境界値の正常ケース）
  it('quantity=1は成功', () => {
    expect(() => OrderItem.create(1)).not.toThrow();
  });
});

// === サバイバー2: 論理演算子 ===
// 元コード
class Order {
  canCancel(): boolean {
    return this._status === 'draft' || this._status === 'pending_payment';
    // Strykerが || を && に変更 → テストが両方の条件を個別検証していなかった
  }
}

// 追加すべきテスト
describe('Order.canCancel', () => {
  it('draft状態はキャンセル可能', () => {
    const order = buildOrder({ status: 'draft' });
    expect(order.canCancel()).toBe(true);
  });

  it('pending_payment状態はキャンセル可能', () => {
    const order = buildOrder({ status: 'pending_payment' });
    expect(order.canCancel()).toBe(true);
  });

  it('completed状態はキャンセル不可', () => {
    const order = buildOrder({ status: 'completed' });
    expect(order.canCancel()).toBe(false);
  });
});

// === 等価ミュータント（スキップが正当） ===
class Money {
  multiply(factor: number): Money {
    // Stryker disable next-line arithmetic
    // 理由: amount * factor と factor * amount は数学的に等価
    return Money.of(this.amount * factor, this.currency);
  }
}
```

```typescript
// CIパイプライン統合 (.github/workflows/mutation-test.yml)
/*
name: Mutation Tests

on:
  push:
    branches: [main]
  pull_request:
    paths:
      - 'src/domain/**'
      - 'src/application/**'

jobs:
  mutation-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - name: Run Stryker
        run: npx stryker run
        env:
          STRYKER_DASHBOARD_API_KEY: ${{ secrets.STRYKER_DASHBOARD_API_KEY }}
      - name: Upload mutation report
        uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: reports/mutation/
*/

// package.json スクリプト
/*
{
  "scripts": {
    "test:mutation": "stryker run",
    "test:mutation:domain": "stryker run --mutate 'src/domain/**/*.ts'",
    "test:mutation:incremental": "stryker run --incremental"
  }
}
*/

// インクリメンタルモード（変更ファイルのみミューテーション）
// stryker.config.ts に追加:
/*
{
  incremental: true,
  incrementalFile: '.stryker-tmp/incremental.json'
}
*/
```

```typescript
// ミューテーションスコアを継続監視するレポート生成

// scripts/mutation-report-summary.ts
import mutationReport from '../reports/mutation/mutation-report.json';

const { metrics } = mutationReport;
const score = metrics.mutationScore;

console.log(`
Mutation Score: ${score.toFixed(1)}%
  Killed:       ${metrics.killed} (検出できたミュータント)
  Survived:     ${metrics.survived} (検出できなかったミュータント ← 要対応)
  No Coverage:  ${metrics.noCoverage} (テストが通らなかったミュータント)
  Timeout:      ${metrics.timeout}
  Total:        ${metrics.total}
`);

// サバイバーの詳細（上位10件）
const survivors = mutationReport.files
  .flatMap(file => file.mutants.filter(m => m.status === 'Survived'))
  .slice(0, 10);

console.log('Top Survivors (要追加テスト):');
survivors.forEach(s => {
  console.log(`  ${s.location.start.line}:${s.location.start.column} ${s.mutatorName}: ${s.description}`);
});

// 目標未達成ならCIを失敗させる
if (score < 80) {
  process.exit(1);
}
```

---

## まとめ

Claude Codeでミューテーションテストを設計する：

1. **CLAUDE.md** にカバレッジはコード実行を計測・ミューテーションスコアはテスト品質を計測・ドメイン層90%+を目標・インフラ層は除外（ROI低）を明記
2. **サバイバー分析でテストの穴を発見** ——`quantity <= 0`を`quantity < 0`に変えてもテストが落ちない場合、`quantity=0`のテストが不足している証拠。Strykerが具体的に「どこのテストが不十分か」を教えてくれる
3. **等価ミュータントは`// Stryker disable`でスキップ** ——`a * b`を`b * a`に変えても結果は同じ（等価ミュータント）。これをスキップすることでスコアが正確になり、本当に問題のある箇所に集中できる
4. **インクリメンタルモード** で変更ファイルのみミューテーション——全ファイルのミューテーションは数時間かかる場合がある。PRでは変更された`src/domain/`ファイルのみを対象にすることでCIを数分に抑えられる

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
