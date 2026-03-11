---
title: "第2章 認証・認可 — JWTとRBACをClaude Codeで実装する"
---

# 第2章 認証・認可 — JWTとRBACをClaude Codeで実装する

認証・認可はバックエンドで最も間違えやすい部分だ。実装ミスがセキュリティインシデントに直結する。

Claude Codeで実装する際は、セキュリティのベストプラクティスを明示的に指示することが重要。

---

## 2-1. JWTの安全な実装

JWTの実装でよくあるミスをClaude Codeに回避させる:

```bash
claude "src/auth/ にJWT認証を実装。

要件:
- access token: 15分有効（httpOnly cookieではなく、Authorizationヘッダー）
- refresh token: 7日有効、httpOnly cookieで保存
- アルゴリズム: RS256（非対称キー）
- refresh tokenのローテーション: 使用ごとに新しいトークンを発行
- リボケーション: Redis でブラックリスト管理

避けること:
- HS256（対称キーは漏洩リスク高）
- JWTをlocalStorageに保存させる実装
- expiry なしのJWT
- 弱いシークレット"
```

実装のポイント:

```typescript
// src/auth/jwt.service.ts
import { SignJWT, jwtVerify } from 'jose'
import { createPrivateKey, createPublicKey } from 'crypto'

export class JwtService {
  private privateKey
  private publicKey

  constructor(private redis: Redis) {
    // 環境変数から鍵を読み込む
    this.privateKey = createPrivateKey(process.env.JWT_PRIVATE_KEY!)
    this.publicKey = createPublicKey(process.env.JWT_PUBLIC_KEY!)
  }

  async signAccessToken(userId: string, roles: string[]): Promise<string> {
    return new SignJWT({ userId, roles })
      .setProtectedHeader({ alg: 'RS256' })
      .setIssuedAt()
      .setExpirationTime('15m')
      .sign(this.privateKey)
  }

  async verifyAccessToken(token: string) {
    // ブラックリストチェック
    const isRevoked = await this.redis.get(`revoked:${token}`)
    if (isRevoked) throw new Error('Token revoked')

    const { payload } = await jwtVerify(token, this.publicKey)
    return payload
  }
}
```

---

## 2-2. RBACの実装

Role-Based Access Control（RBAC）をシンプルに実装:

```bash
claude "Expressミドルウェアとして以下のRBACを実装:

ロール: admin, manager, user（階層構造）
- admin は全リソースにアクセス可能
- manager はチームリソースにアクセス可能
- user は自分のリソースのみアクセス可能

使い方:
router.get('/admin/stats', authenticate, authorize('admin'), handler)
router.get('/teams/:id', authenticate, authorize('manager', 'admin'), handler)
router.get('/profile', authenticate, authorize('user'), handler)

リソースオーナーチェック（自分のリソースのみ）も実装"
```

実装例:

```typescript
// src/auth/rbac.middleware.ts
export const authorize = (...allowedRoles: Role[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user?.role

    if (!userRole || !allowedRoles.includes(userRole)) {
      throw new AppError('FORBIDDEN', 'アクセス権限がありません', 403)
    }

    next()
  }
}

// リソースオーナーチェック
export const authorizeOwner = (getResourceOwnerId: (req: Request) => Promise<string>) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    const ownerId = await getResourceOwnerId(req)

    if (req.user?.role === 'admin') return next()
    if (req.user?.userId !== ownerId) {
      throw new AppError('FORBIDDEN', 'このリソースへのアクセス権限がありません', 403)
    }

    next()
  }
}
```

---

## 2-3. レート制限とブルートフォース対策

```bash
claude "ログインエンドポイントにブルートフォース対策を追加:

要件:
- IPアドレスごとに5回失敗したら15分間ロック
- アカウントごとに10回失敗したら24時間ロック
- Redisでカウント管理
- ロック解除のログを記録"
```

---

## 2-4. OAuth2との統合

Google/GitHubなどのOAuth2プロバイダーとの統合:

```bash
claude "Passport.jsを使わずにGoogle OAuth2を実装:

フロー:
1. GET /auth/google → Google OAuth URLにリダイレクト
2. GET /auth/google/callback → アクセストークン取得
3. ユーザー情報取得 → DB照合 or 新規作成
4. JWTを発行して返す

セキュリティ:
- state parameterでCSRF対策
- PKCEを使用（推奨）
- nonce検証"
```

---

## まとめ

認証・認可の実装でClaude Codeに指示する際のポイント:

- **セキュリティ要件を明示する**: 「安全に実装して」ではなく「RS256を使い、refresh tokenはhttpOnly cookieで」と具体的に
- **アンチパターンを明示する**: 「HS256は使わない」「localStorageに保存しない」
- **テストも一緒に**: 認証ロジックは必ずユニットテストを書かせる

次の章では、DBの設計と最適化を扱う。
