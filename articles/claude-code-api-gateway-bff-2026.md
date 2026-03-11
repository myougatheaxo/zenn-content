---
title: "Claude CodeでAPI GatewayとBFFを設計する：フロントエンド向け最適化レイヤー"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "api", "architecture"]
published: true
---

## はじめに

フロントエンドがマイクロサービスを直接呼ぶと、複数のAPIリクエストでWaterfall問題が発生する。BFF（Backend for Frontend）で集約してフロントエンドに最適なレスポンスを返す。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにBFF設計ルールを書く

```markdown
## API Gateway / BFF設計ルール

### BFF責務
- 複数サービスのデータを集約（N→1リクエスト）
- フロントエンドに必要なフィールドのみを返す（over-fetch防止）
- 認証・認可をここで一元管理
- レスポンスキャッシュ（Redis TTL 1-5分）

### 集約戦略
- 並列実行: 依存関係のないサービスはPromise.all
- 直列実行: 依存関係がある場合のみ直列
- 部分失敗: 1つのサービスが失敗しても他のデータは返す

### セキュリティ
- フロントエンドからの直接マイクロサービスアクセス禁止
- サービス間通信はmTLS または API Key（内部のみ）
- レートリミットはBFF層で実装
```

---

## BFFの生成

```
ダッシュボード画面向けのBFFエンドポイントを設計してください。

必要なデータ:
- ユーザープロフィール（UserService）
- 最近の注文5件（OrderService）
- 未読通知数（NotificationService）

要件：
- 3サービスを並列で呼び出して集約
- 1サービスが失敗しても他のデータは返す
- Redisでレスポンスをキャッシュ（TTL 2分）

生成ファイル: src/bff/dashboardBff.ts
```

---

## 生成されるBFF実装

```typescript
// src/bff/dashboardBff.ts
interface DashboardResponse {
  user: UserProfile | null;
  recentOrders: Order[];
  unreadNotificationCount: number;
  errors: string[];
}

async function fetchUserProfile(userId: string): Promise<UserProfile> {
  const res = await fetch(`${process.env.USER_SERVICE_URL}/users/${userId}`, {
    headers: { 'X-Internal-Token': process.env.INTERNAL_API_KEY! },
  });
  if (!res.ok) throw new Error(`UserService: ${res.status}`);
  return res.json();
}

async function fetchRecentOrders(userId: string): Promise<Order[]> {
  const res = await fetch(
    `${process.env.ORDER_SERVICE_URL}/orders?userId=${userId}&limit=5`,
    { headers: { 'X-Internal-Token': process.env.INTERNAL_API_KEY! } }
  );
  if (!res.ok) throw new Error(`OrderService: ${res.status}`);
  const data = await res.json();
  return data.orders;
}

async function fetchUnreadCount(userId: string): Promise<number> {
  const res = await fetch(
    `${process.env.NOTIFICATION_SERVICE_URL}/notifications/unread-count?userId=${userId}`,
    { headers: { 'X-Internal-Token': process.env.INTERNAL_API_KEY! } }
  );
  if (!res.ok) throw new Error(`NotificationService: ${res.status}`);
  const data = await res.json();
  return data.count;
}

export async function getDashboardData(userId: string): Promise<DashboardResponse> {
  const cacheKey = `bff:dashboard:${userId}`;

  // キャッシュチェック
  const cached = await redis.get(cacheKey);
  if (cached) {
    logger.debug({ userId, cacheHit: true }, 'BFF cache hit');
    return JSON.parse(cached);
  }

  // 3サービスを並列で呼び出し（Promise.allSettled で部分失敗を許容）
  const [userResult, ordersResult, notificationsResult] = await Promise.allSettled([
    fetchUserProfile(userId),
    fetchRecentOrders(userId),
    fetchUnreadCount(userId),
  ]);

  const errors: string[] = [];

  const response: DashboardResponse = {
    user: userResult.status === 'fulfilled' ? userResult.value : null,
    recentOrders: ordersResult.status === 'fulfilled' ? ordersResult.value : [],
    unreadNotificationCount:
      notificationsResult.status === 'fulfilled' ? notificationsResult.value : 0,
    errors,
  };

  // 失敗したサービスのエラーを収集（クライアントに部分失敗を伝える）
  if (userResult.status === 'rejected') errors.push('user_service_unavailable');
  if (ordersResult.status === 'rejected') errors.push('order_service_unavailable');
  if (notificationsResult.status === 'rejected') errors.push('notification_service_unavailable');

  // エラーなしの場合のみキャッシュ（部分失敗データはキャッシュしない）
  if (errors.length === 0) {
    await redis.set(cacheKey, JSON.stringify(response), { EX: 120 }); // 2分
  }

  return response;
}
```

```typescript
// src/routes/bff.ts
router.get('/bff/dashboard', authenticate, async (req, res) => {
  const data = await getDashboardData(req.user.id);

  // 部分失敗は200で返す（フロントエンドが判断）
  res.status(200).json(data);
});
```

---

## タイムアウト付きフェッチ

```typescript
// サービス呼び出しに必ずタイムアウトを設定
async function fetchWithTimeout<T>(
  url: string,
  options: RequestInit,
  timeoutMs = 3000
): Promise<T> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const res = await fetch(url, { ...options, signal: controller.signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  } finally {
    clearTimeout(timeout);
  }
}
```

---

## まとめ

Claude CodeでBFFを設計する：

1. **CLAUDE.md** に集約責務・並列実行・部分失敗許容・キャッシュTTLを明記
2. **Promise.allSettled** で部分失敗を許容（1サービス障害でダッシュボード全体が落ちない）
3. **エラー情報をレスポンスに含める** でフロントエンドが劣化表示を実装できる
4. **Redis キャッシュ** でバックエンドサービスへの負荷を削減

---

*BFF/API Gateway設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
