---
title: "Claude Codeで安全な認証を実装する：JWT・OAuth・セッションの設計パターン"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "security", "jwt", "oauth", "認証"]
published: true
---

## はじめに

認証コードのミスは深刻なセキュリティ問題になる。Claude Codeは便利だが、制約なしに認証を実装させると、`localStorage`にJWTを保存したり、bcryptのコストファクターが低すぎたりする問題が混入する。

CLAUDE.mdで正しいパターンを強制する。

---

## CLAUDE.mdに認証ルールを書く

```markdown
## 認証セキュリティルール

### JWT設計
- アルゴリズム: RS256（本番）、HS256（開発のみ）
- アクセストークン: 15分有効期限
- リフレッシュトークン: 7日間、Redisに保存（失効可能にするため）
- ストレージ: アクセストークンはメモリ、リフレッシュはhttpOnly Cookie
- **禁止**: トークンをlocalStorageに保存
- ペイロード: userIdとroleのみ（機密情報・パスワードハッシュ等は禁止）

### パスワード
- ハッシュ: bcrypt、コストファクター12
- ログへの出力禁止（平文/ハッシュ両方）
- 比較: bcrypt.compare()を使う（文字列直接比較禁止）
- 最低要件: 8文字以上、大文字+数字を含む

### セッション
- ストア: Redis（インメモリ or DBセッション禁止）
- Cookie設定: httpOnly, secure, sameSite: strict
- ログイン時にsession.regenerate()を呼ぶ

### OAuth
- stateパラメータをセッションと照合して検証（CSRF対策必須）
- PKCE: パブリッククライアント（SPA/モバイル）では必須
- 固有識別子: プロバイダーのユーザーID（メールアドレスは変更可能のため禁止）
- メールが一致する既存ユーザーへのアカウントリンクは手動確認必須

### レート制限
- ログインエンドポイント: 10回/分/IP
- ユーザー登録: 5回/分/IP
- パスワードリセット: 3回/時/IP
```

---

## JWT認証の実装パターン

```
CLAUDE.mdの認証ルールに従って、JWTベースの認証システムを実装してください。

必要な機能：
- login(): 認証 → アクセストークン(15分) + リフレッシュトークン(7日)を発行
- refreshToken(): リフレッシュトークンをRedisで検証 → 新アクセストークン
- logout(): Redisのリフレッシュトークンを削除
- authMiddleware: Authorizationヘッダーからトークンを取得・検証

RS256を使い、秘密鍵はsrc/config/keys.tsから読む。
```

---

## 認証Middlewareの生成

```
JWT検証Middlewareを生成してください。

要件：
- Authorization: Bearer <token> からトークンを取得
- RS256公開鍵で署名を検証
- 失敗ケースと対応するコード:
  - トークンなし: 401 + MISSING_TOKEN
  - 無効な署名: 401 + INVALID_TOKEN
  - 期限切れ: 401 + EXPIRED_TOKEN
- 成功時: デコードしたユーザー情報をreq.userに付与

保存先：src/middleware/auth.ts
```

---

## OAuthセキュリティ

```
Google OAuthのコールバックハンドラーを生成してください。

セキュリティ要件（CLAUDE.mdに従う）：
- stateパラメータをセッションと照合
- 認可コードフロー（サーバーサイドでコード交換）
- ユーザーIDはGoogleのsub（メールアドレスは禁止）
- 既存ユーザー: email一致でもOAuthリンクには手動確認を要求

保存先：src/routes/auth/google.ts
```

---

## よくある認証ミスとCLAUDE.mdによる防止

| ミス | 発生する問題 | CLAUDE.mdで防ぐ |
|------|------|------|
| localStorage にJWT保存 | XSS脆弱性 | httpOnly Cookie必須 |
| ペイロードに機密情報 | トークン漏洩 | payload制限ルール |
| bcryptコストファクター10 | ブルートフォース耐性低 | コストファクター12指定 |
| OAuth stateなし | CSRF攻撃 | state検証必須 |
| メールアドレスで一意識別 | アカウント乗っ取り | providerのユーザーID使用 |

---

## 認証テストの生成

```
JWT認証Middlewareのテストを生成してください。

テストケース：
- 有効なトークン: 200 + req.userにユーザー情報が付与される
- Authorizationヘッダーなし: 401 + MISSING_TOKEN
- 無効な署名のトークン: 401 + INVALID_TOKEN
- 期限切れのトークン: 401 + EXPIRED_TOKEN
- 間違ったaudienceのトークン: 401

テストではJWT署名の実際の鍵を使わない（jwtモジュールをモック）。
```

---

## まとめ

Claude Codeで安全な認証実装：

1. **CLAUDE.md** に具体的なセキュリティルールを書く
2. **JWT**: RS256 + 短い有効期限 + httpOnly Cookieストレージ
3. **パスワード**: bcrypt コストファクター12
4. **OAuth**: state検証必須 + providerのユーザーID使用
5. **テスト**: 全ての失敗ケースを網羅

---

*認証コードのセキュリティレビューは **Security Pack（¥1,480）** の `/code-review` で自動化できます。PromptWorksで購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
