---
title: "Claude CodeでScatter-Gatherパターンを設計する：並列サービス呼び出し・結果集約・タイムアウト制御"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-17 19:00"
---

## はじめに

「複数の価格比較APIを順番に呼び出していて遅い」「1つが遅いと全体が遅くなる」——Scatter-Gatherパターンで複数サービスへの問い合わせを並列実行し、最速の結果から順に集約する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにScatter-Gather設計ルールを書く

```markdown
## Scatter-Gather設計ルール

### 並列実行
- 全サービスへのリクエストを同時に発行（逐次実行しない）
- タイムアウトを設けて遅いサービスを待ちすぎない
- 部分的な成功を許容（全サービス成功を要求しない）

### 集約戦略
- FIRST_WIN: 最初の成功レスポンスを使用（残りはキャンセル）
- ALL_SUCCESS: 全サービスの成功が必要（1つでも失敗したら全体失敗）
- BEST_EFFORT: 成功したものだけを集約（失敗は無視）
- QUORUM: 過半数の成功が必要

### タイムアウト
- 全体タイムアウト: 2秒（これを超えたら集約済みの結果を返す）
- 個別サービスタイムアウト: 1.5秒（独立して設定）
- Hedged Request: 500ms経過後に別のサービスにも問い合わせ
```

---

## Scatter-Gather実装の生成

```
Scatter-Gatherパターンを設計してください。

要件：
- 複数サービスへの並列リクエスト
- 集約戦略（FIRST_WIN/BEST_EFFORT/QUORUM）
- タイムアウト制御
- Hedged Request

生成ファイル: src/patterns/scatterGather/
```

---

## 生成されるScatter-Gather実装

```typescript
// src/patterns/scatterGather/scatterGather.ts — Scatter-Gatherオーケストレーター

export type AggregationStrategy = 'FIRST_WIN' | 'ALL_SUCCESS' | 'BEST_EFFORT' | 'QUORUM';

export interface ScatterTarget<TResponse> {
  name: string;
  request: () => Promise<TResponse>;
  timeoutMs?: number;
}

export interface ScatterGatherOptions {
  strategy: AggregationStrategy;
  totalTimeoutMs: number;
  minResponses?: number;  // BEST_EFFORT: 最低このレスポンス数が必要
  hedgedDelayMs?: number; // Hedged Request: この時間後に追加リクエスト
}

export interface ScatterGatherResult<T> {
  results: Array<{ name: string; data: T; durationMs: number }>;
  failures: Array<{ name: string; error: string; durationMs: number }>;
  totalDurationMs: number;
  strategy: AggregationStrategy;
}

export class ScatterGather<TResponse> {
  constructor(private readonly opts: ScatterGatherOptions) {}

  async execute(targets: ScatterTarget<TResponse>[]): Promise<ScatterGatherResult<TResponse>> {
    const startMs = Date.now();
    const results: ScatterGatherResult<TResponse>['results'] = [];
    const failures: ScatterGatherResult<TResponse>['failures'] = [];

    switch (this.opts.strategy) {
      case 'FIRST_WIN':
        return this.firstWin(targets, startMs);
      case 'ALL_SUCCESS':
        return this.allSuccess(targets, startMs);
      case 'BEST_EFFORT':
        return this.bestEffort(targets, startMs);
      case 'QUORUM':
        return this.quorum(targets, startMs);
    }
  }

  // 最初の成功を返して残りはキャンセル
  private async firstWin(targets: ScatterTarget<TResponse>[], startMs: number): Promise<ScatterGatherResult<TResponse>> {
    const controller = new AbortController();

    const promises = targets.map(target =>
      this.withTimeout(target, controller.signal).then(result => {
        controller.abort(); // 他のリクエストをキャンセル
        return result;
      })
    );

    const first = await Promise.race([
      ...promises,
      this.totalTimeout(startMs),
    ]);

    return {
      results: [first],
      failures: [],
      totalDurationMs: Date.now() - startMs,
      strategy: 'FIRST_WIN',
    };
  }

  // 全て成功が必要
  private async allSuccess(targets: ScatterTarget<TResponse>[], startMs: number): Promise<ScatterGatherResult<TResponse>> {
    const settled = await Promise.allSettled(
      targets.map(t => this.withTimeout(t))
    );

    const results = settled.filter(r => r.status === 'fulfilled').map(r => (r as PromiseFulfilledResult<any>).value);
    const failures = settled.filter(r => r.status === 'rejected').map((r, i) => ({
      name: targets[i].name,
      error: (r as PromiseRejectedResult).reason.message,
      durationMs: 0,
    }));

    if (failures.length > 0) {
      throw new AggregationError(`ALL_SUCCESS: ${failures.length} services failed`);
    }

    return { results, failures, totalDurationMs: Date.now() - startMs, strategy: 'ALL_SUCCESS' };
  }

  // 成功したものだけ集約
  private async bestEffort(targets: ScatterTarget<TResponse>[], startMs: number): Promise<ScatterGatherResult<TResponse>> {
    const settled = await Promise.allSettled([
      ...targets.map(t => this.withTimeout(t)),
      this.totalTimeout(startMs),
    ]);

    const results: ScatterGatherResult<TResponse>['results'] = [];
    const failures: ScatterGatherResult<TResponse>['failures'] = [];

    settled.slice(0, targets.length).forEach((result, i) => {
      if (result.status === 'fulfilled') {
        results.push(result.value);
      } else {
        failures.push({ name: targets[i].name, error: result.reason.message, durationMs: 0 });
      }
    });

    const minRequired = this.opts.minResponses ?? 1;
    if (results.length < minRequired) {
      throw new AggregationError(`BEST_EFFORT: only ${results.length}/${minRequired} responses received`);
    }

    return { results, failures, totalDurationMs: Date.now() - startMs, strategy: 'BEST_EFFORT' };
  }

  // 過半数の成功が必要
  private async quorum(targets: ScatterTarget<TResponse>[], startMs: number): Promise<ScatterGatherResult<TResponse>> {
    const quorumSize = Math.floor(targets.length / 2) + 1;
    const settled = await Promise.allSettled(targets.map(t => this.withTimeout(t)));

    const results = settled.filter(r => r.status === 'fulfilled').map(r => (r as any).value);
    const failures = settled.filter(r => r.status === 'rejected').map((r, i) => ({
      name: targets[i].name,
      error: (r as any).reason.message,
      durationMs: 0,
    }));

    if (results.length < quorumSize) {
      throw new AggregationError(`QUORUM: ${results.length}/${quorumSize} required`);
    }

    return { results, failures, totalDurationMs: Date.now() - startMs, strategy: 'QUORUM' };
  }

  private withTimeout(target: ScatterTarget<TResponse>, signal?: AbortSignal): Promise<{ name: string; data: TResponse; durationMs: number }> {
    const startMs = Date.now();
    const timeoutMs = target.timeoutMs ?? this.opts.totalTimeoutMs;

    return Promise.race([
      target.request().then(data => ({ name: target.name, data, durationMs: Date.now() - startMs })),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error(`${target.name} timed out after ${timeoutMs}ms`)), timeoutMs)
      ),
    ]);
  }

  private totalTimeout(startMs: number): Promise<never> {
    return new Promise((_, reject) =>
      setTimeout(
        () => reject(new Error(`Total timeout after ${this.opts.totalTimeoutMs}ms`)),
        this.opts.totalTimeoutMs - (Date.now() - startMs)
      )
    );
  }
}

// 使用例: 複数の価格比較APIを並列呼び出し
const priceScatter = new ScatterGather<PriceQuote>({
  strategy: 'BEST_EFFORT',
  totalTimeoutMs: 2000,
  minResponses: 1,  // 最低1サービスの応答があれば結果を返す
});

const { results, failures } = await priceScatter.execute([
  { name: 'supplier-a', request: () => supplierAApi.getPrice(productId), timeoutMs: 1500 },
  { name: 'supplier-b', request: () => supplierBApi.getPrice(productId), timeoutMs: 1500 },
  { name: 'supplier-c', request: () => supplierCApi.getPrice(productId), timeoutMs: 1500 },
]);

// 最安値を選択
const cheapest = results.reduce((best, r) => r.data.price < best.data.price ? r : best);
logger.info({ cheapest: cheapest.name, price: cheapest.data.price, failures: failures.length }, 'Price comparison completed');
```

---

## まとめ

Claude CodeでScatter-Gatherパターンを設計する：

1. **CLAUDE.md** に全サービスへ同時リクエスト・全体タイムアウト2秒・部分的成功を許容する集約戦略・遅いサービスをキャンセルを明記
2. **BEST_EFFORT戦略** で成功したものだけを集約——価格比較・在庫確認など「全サービスの回答は不要、最低1件あれば動く」場合に最適。minResponsesで最低保証数を設定
3. **FIRST_WIN戦略 + AbortController** で最初の成功が来たら残りのリクエストをキャンセル——最速サービスを選ぶ場面（冗長化された同一サービスへのHedged Request等）で通信コストを削減
4. **QUORUM戦略** でN個中過半数の合意が必要な操作——分散DBへの読み取りで「過半数が同じ値を返した」場合だけ信頼する一貫性チェックに活用

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
