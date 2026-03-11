---
title: "Claude CodeでAPIレート制限を実装する：Redis + Sliding Window"
emoji: "⏱️"
type: "tech"
topics: ["claudecode", "redis", "nodejs", "api", "セキュリティ"]
published: true
---

## はじめに

レート制限のないAPIは、ある日突然あなたのインフラを破壊します。

- **DDoS**: 悪意ある攻撃者が秒間数千リクエストを送り込む
- **コスト爆発**: OpenAI APIやStripe APIを叩くバックエンドが、バグったクライアントに呼ばれ続けて請求が跳ね上がる
- **公平性の崩壊**: 1ユーザーが帯域を独占し、他のユーザーが応答を受け取れなくなる

Claude Codeでレート制限を実装するとき、曖昧な指示では「なんとなく動く」コードが生成されます。CLAUDE.mdにルールを明文化することで、全エンドポイントに一貫した実装を強制できます。

---

## CLAUDE.mdにレート制限ルールを書く

```markdown
## APIレート制限ルール

### 必須要件
- 全パブリックAPIエンドポイントにレート制限を実装すること
- レート制限の実装はRedis + Sliding Window方式を使用する
- Fixed Windowは使用禁止（バースト攻撃に脆弱なため）

### 制限単位
- 認証済みユーザー: user_id単位で制限
- 未認証リクエスト: IPアドレス単位で制限
- Redisキー形式: `rl:{identifier}:{endpoint}` (例: `rl:user_123:api/posts`)

### デフォルト制限値
- 読み取り系 (GET): 100リクエスト/分
- 書き込み系 (POST/PUT/PATCH): 20リクエスト/分
- 削除 (DELETE): 10リクエスト/分
- 認証エンドポイント (/auth/login, /auth/register): 5リクエスト/分

### レスポンスヘッダー（必須）
- X-RateLimit-Limit: 制限値
- X-RateLimit-Remaining: 残りリクエスト数
- X-RateLimit-Reset: リセット時刻（Unix timestamp）
- Retry-After: 制限超過時のみ（秒数）

### 制限超過時
- HTTPステータス: 429 Too Many Requests
- レスポンスボディ: { "error": "rate_limit_exceeded", "retryAfter": N }
```

---

## Sliding Windowレート制限の実装プロンプト

```
RedisのSorted Setを使ったSliding Windowレート制限ミドルウェアを実装してください。

要件:
- TypeScript + Express + ioredis
- CLAUDE.mdのレート制限ルールに従う
- Sliding Window方式（Fixed Windowは使用しない）
- ユーザーIDが取得できる場合はuser_id、できない場合はIPアドレスで制限
- 全レスポンスにX-RateLimit-*ヘッダーを付与
- 429時はRetry-Afterヘッダーも付与

Redisキー設計:
- キー: rl:{identifier}:{endpoint_group}
- 値: Sorted Set（スコア=タイムスタンプ、メンバー=リクエストID）
- TTL: ウィンドウサイズ（秒）

エンドポイントグループの判定:
- /auth/* → authグループ（5req/min）
- GET /api/* → read グループ（100req/min）
- POST|PUT|PATCH /api/* → write グループ（20req/min）
- DELETE /api/* → delete グループ（10req/min）
```

期待される出力（抜粋）:

```typescript
// middleware/rateLimiter.ts
import { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';
import { v4 as uuidv4 } from 'uuid';

const redis = new Redis(process.env.REDIS_URL);

type LimitGroup = 'auth' | 'read' | 'write' | 'delete';

const LIMITS: Record<LimitGroup, { max: number; windowSec: number }> = {
  auth:   { max: 5,   windowSec: 60 },
  read:   { max: 100, windowSec: 60 },
  write:  { max: 20,  windowSec: 60 },
  delete: { max: 10,  windowSec: 60 },
};

function getGroup(req: Request): LimitGroup {
  if (req.path.startsWith('/auth/')) return 'auth';
  if (req.method === 'DELETE') return 'delete';
  if (['POST', 'PUT', 'PATCH'].includes(req.method)) return 'write';
  return 'read';
}

function getIdentifier(req: Request): string {
  const userId = (req as any).user?.id;
  const ip = req.ip ?? req.socket.remoteAddress ?? 'unknown';
  return userId ? `user_${userId}` : `ip_${ip}`;
}

export async function rateLimiter(
  req: Request, res: Response, next: NextFunction
) {
  const group = getGroup(req);
  const { max, windowSec } = LIMITS[group];
  const identifier = getIdentifier(req);
  const key = `rl:${identifier}:${group}`;
  const now = Date.now();
  const windowStart = now - windowSec * 1000;

  // Sliding window: 古いエントリを削除 → 追加 → カウント
  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, windowStart);
  pipeline.zadd(key, now, `${now}-${uuidv4()}`);
  pipeline.zcard(key);
  pipeline.expire(key, windowSec);
  const results = await pipeline.exec();

  const count = results?.[2]?.[1] as number ?? 0;
  const remaining = Math.max(0, max - count);
  const resetAt = Math.ceil((now + windowSec * 1000) / 1000);

  res.setHeader('X-RateLimit-Limit', max);
  res.setHeader('X-RateLimit-Remaining', remaining);
  res.setHeader('X-RateLimit-Reset', resetAt);

  if (count > max) {
    const retryAfter = Math.ceil(windowSec - (now - windowStart) / 1000);
    res.setHeader('Retry-After', retryAfter);
    return res.status(429).json({
      error: 'rate_limit_exceeded',
      retryAfter,
    });
  }

  next();
}
```

---

## レート制限ヘッダーの設定プロンプト

```
実装済みのrateLimiterミドルウェアに対して、以下を確認・追加してください:

1. X-RateLimit-Limit: 現在のエンドポイントグループの最大値
2. X-RateLimit-Remaining: 残りリクエスト数（マイナスにならないよう Math.max(0, ...) で保護）
3. X-RateLimit-Reset: Unix timestamp（秒）でリセット時刻
4. Retry-After: 429レスポンス時のみ。秒数（整数）で返す

追加要件:
- 全ての成功レスポンス（2xx）にもX-RateLimit-*ヘッダーを付与
- ヘッダーの型はstring（数値をsetHeaderに渡す場合は自動変換されるが明示的に変換）
- X-RateLimit-Policyヘッダーも追加: "100;w=60" 形式（RFC 6585準拠）
```

---

## エンドポイント別の制限設定プロンプト

認証エンドポイントは特に厳しく制限する必要があります。

```
以下の要件でエンドポイント別レート制限を設定してください:

認証系（最も厳しく）:
- POST /auth/login: 5回/分、失敗時は追加ペナルティ（成功したらカウントリセット）
- POST /auth/register: 3回/時間
- POST /auth/forgot-password: 3回/時間、同一メールアドレスで

API系（機能別）:
- GET /api/search: 30回/分（重いクエリのため）
- POST /api/export: 5回/時間（バックグラウンドジョブのため）
- GET /api/* (その他): 100回/分
- POST|PUT|PATCH /api/*: 20回/分

管理者エンドポイント:
- /admin/*: 認証必須 + 200回/分（管理者ロールで緩和）

実装方針:
- エンドポイントパターンマッチングで自動グループ判定
- 特定エンドポイントは個別設定でオーバーライド可能
- 管理者ロールの判定は req.user.role === 'admin' で行う
```

実装イメージ:

```typescript
// エンドポイント個別設定
const ENDPOINT_OVERRIDES: Record<string, Partial<LimitConfig>> = {
  'POST:/auth/login':            { max: 5,  windowSec: 60  },
  'POST:/auth/register':         { max: 3,  windowSec: 3600 },
  'POST:/auth/forgot-password':  { max: 3,  windowSec: 3600 },
  'GET:/api/search':             { max: 30, windowSec: 60  },
  'POST:/api/export':            { max: 5,  windowSec: 3600 },
};

function resolveLimit(req: Request): LimitConfig {
  const key = `${req.method}:${req.path}`;
  if (ENDPOINT_OVERRIDES[key]) {
    return { ...LIMITS[getGroup(req)], ...ENDPOINT_OVERRIDES[key] };
  }
  // 管理者は制限緩和
  if (req.path.startsWith('/admin/') && (req as any).user?.role === 'admin') {
    return { max: 200, windowSec: 60 };
  }
  return LIMITS[getGroup(req)];
}
```

---

## まとめ

CLAUDE.mdにレート制限ルールを明文化することで:

1. **実装の一貫性**: 新しいエンドポイントを追加するたびに「レート制限ってどうするんだっけ」と悩まなくなる
2. **コードレビューの効率化**: 「CLAUDE.mdの制限値に合ってる？」の一言でチェックポイントが明確になる
3. **Security by Design**: 後付けではなく、設計段階からセキュリティが組み込まれる

Sliding Window方式はFixed Windowより実装コストが若干高いですが、バースト攻撃への耐性が大幅に向上します。RedisのSorted Setを使えば、O(log N)の計算量で正確なウィンドウ管理が可能です。

---

**Security Pack**（¥1,480）の `/security-check` コマンドで、レート制限の設定漏れ・ヘッダー不足・Fixed Window使用を自動検出できます。

👉 https://prompt-works.jp

*Myouga (@myougaTheAxo) — ウーパールーパーVTuber、セキュリティ重視のClaude Codeエンジニア*
