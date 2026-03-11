---
title: "Claude Codeでマルチリージョンデータ同期を設計する：地理的レプリケーション・競合解決・CRDTs"
emoji: "🌏"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "redis"]
published: true
published_at: "2026-03-16 23:00"
---

## はじめに

「東京とUSのデータが違う」「マルチリージョン展開で書き込み競合が発生している」——地理的に分散したデータベースの同期・競合解決・CRDTs（競合のない分散データ型）をClaude Codeに設計させる。

---

## CLAUDE.mdにマルチリージョン設計ルールを書く

```markdown
## マルチリージョンデータ同期設計ルール

### レプリケーション方式
- Active-Passive: 1リージョンが書き込み、残りは読み取りレプリカ（シンプル）
- Active-Active: 全リージョンが書き込み可能（高可用性だが競合が発生）
- 本システム: Active-Passive（書き込みは東京、US/EUは読み取りのみ）

### 競合解決（Active-Activeの場合）
- Last-Write-Wins（LWW）: タイムスタンプが新しい方を採用
- CRDT: カウンター・セット・マップは数学的に競合なしにマージ可能
- アプリケーションレベル解決: ユーザーに競合を通知して選択させる

### レプリカラグ対策
- 書き込み後の読み取りは書き込みリージョンへルーティング
- ラグモニタリング: pg_stat_replicationで遅延を監視
```

---

## 生成されるマルチリージョン実装（抜粋）

```typescript
// リージョンルーティング（書き込みはプライマリへ）
export class RegionRouter {
  private readonly primaryRegion = process.env.PRIMARY_REGION ?? 'ap-northeast-1';
  private readonly currentRegion = process.env.AWS_REGION ?? 'ap-northeast-1';

  isPrimary(): boolean { return this.currentRegion === this.primaryRegion; }

  async routeWrite<T>(fn: () => Promise<T>): Promise<T> {
    if (this.isPrimary()) return fn();

    // 非プライマリリージョン: プライマリへリダイレクト
    throw new CrossRegionRedirectError(this.primaryRegion, 'Write operations must target primary region');
  }

  async routeRead<T>(fn: () => Promise<T>, allowStale = false): Promise<T> {
    if (!allowStale) {
      // 強い読み取り一貫性: プライマリから読む
      return this.readFromPrimary(fn);
    }
    return fn(); // ローカルレプリカから読む（高速だが古いデータの可能性）
  }
}

// CRDTカウンター（分散環境で競合なしにインクリメント）
export class CRDTCounter {
  async increment(key: string, amount = 1): Promise<void> {
    const nodeId = process.env.NODE_ID!;
    await redis.hincrby(`crdt:counter:${key}`, nodeId, amount);
  }

  async getValue(key: string): Promise<number> {
    const values = await redis.hgetall(`crdt:counter:${key}`);
    return Object.values(values ?? {}).reduce((sum, v) => sum + parseInt(v), 0);
  }

  // 全リージョンの値をマージ（競合なし: 全ノードの値を合算）
  async merge(key: string, remoteCounters: Record<string, number>): Promise<void> {
    const pipeline = redis.pipeline();
    for (const [nodeId, value] of Object.entries(remoteCounters)) {
      const current = parseInt(await redis.hget(`crdt:counter:${key}`, nodeId) ?? '0');
      pipeline.hset(`crdt:counter:${key}`, nodeId, Math.max(current, value));
    }
    await pipeline.exec();
  }
}

// Last-Write-Wins競合解決
export class LWWConflictResolver {
  async resolve<T extends { updatedAt: Date }>(local: T, remote: T): Promise<T> {
    return local.updatedAt >= remote.updatedAt ? local : remote;
  }
}

// レプリカラグモニタリング
export async function checkReplicationLag(): Promise<{ lagMs: number; status: 'ok' | 'warning' | 'critical' }> {
  const result = await primaryDB.$queryRaw<Array<{ lag_ms: number }>>`
    SELECT EXTRACT(EPOCH FROM replay_lag) * 1000 AS lag_ms
    FROM pg_stat_replication ORDER BY lag_ms DESC LIMIT 1
  `;
  const lagMs = result[0]?.lag_ms ?? 0;
  return {
    lagMs,
    status: lagMs > 30000 ? 'critical' : lagMs > 5000 ? 'warning' : 'ok',
  };
}
```

---

## まとめ

1. **CLAUDE.md** にActive-Passive（書き込みは東京のみ）・CRDTで競合なし加算・LWW（タイムスタンプ）競合解決・pg_stat_replicationでラグ監視を明記
2. **RegionRouter** で書き込みをプライマリへ強制リダイレクト——レプリカへの書き込みで同期が壊れるのを防止
3. **CRDTカウンター** でノードごとのカウンターを合算——どのリージョンで何回インクリメントされても競合なしにマージ
4. **LWW競合解決** でActive-Active時の書き込み競合を最新タイムスタンプで解決——単純だが多くのユースケースで十分

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
