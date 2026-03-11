---
title: "Claude CodeでセキュリティヘッダーをCLAUDE.mdで設計する：Helmet・CSP・HSTS"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "express"]
published: true
---

## はじめに

HTTPヘッダーのデフォルト設定はセキュリティが弱い——XSSが通る、クリックジャッキングされる、HTTPSじゃない通信が通る。Claude CodeにHelmetを使ったセキュリティヘッダー設定を生成させる。

---

## CLAUDE.mdにセキュリティヘッダールールを書く

```markdown
## セキュリティヘッダー設計ルール

### 必須ヘッダー
- Content-Security-Policy (CSP): XSSの緩和（インラインスクリプト禁止）
- Strict-Transport-Security (HSTS): HTTPS強制（max-age=31536000）
- X-Frame-Options: DENY（クリックジャッキング防止）
- X-Content-Type-Options: nosniff（MIMEスニッフィング防止）
- Referrer-Policy: strict-origin-when-cross-origin

### CSP設定
- default-src: 'none'（ホワイトリスト方式）
- script-src: 自ドメインのみ（eval禁止、inline禁止）
- style-src: 自ドメイン + 'unsafe-inline'（CSSフレームワークは許可）
- connect-src: 自APIドメインのみ
- report-uri: CSP違反をエンドポイントで収集

### 開発環境
- CSPのreport-onlyモードで違反を確認してから適用
- localhost向けに一部緩和（websocket等）
```

---

## セキュリティヘッダーの生成

```
Helmetを使ったセキュリティヘッダー設定を生成してください。

環境:
- Express.js
- React SPA（script/style読み込みあり）
- AWS CloudFront経由

要件：
- CSP設定（インラインスクリプト禁止）
- HSTS 1年
- CSP違反レポートエンドポイント
- 開発/本番で設定を切り替え

生成ファイル: src/middleware/security.ts
```

---

## 生成されるセキュリティヘッダー

```typescript
// src/middleware/security.ts
import helmet from 'helmet';

const isDev = process.env.NODE_ENV !== 'production';
const APP_DOMAIN = process.env.APP_URL ?? 'https://example.com';
const API_DOMAIN = process.env.API_URL ?? 'https://api.example.com';

export const securityMiddleware = helmet({
  contentSecurityPolicy: {
    useDefaults: false,
    directives: {
      defaultSrc: ["'none'"],
      scriptSrc: [
        "'self'",
        // Nonceを使う場合（インラインスクリプトを安全に許可）
        // (req, res) => `'nonce-${(res as any).locals.nonce}'`,
        // CDN利用の場合のみ追加
        // 'https://cdn.jsdelivr.net',
      ],
      styleSrc: [
        "'self'",
        "'unsafe-inline'", // CSSフレームワーク（Tailwind等）のため
      ],
      imgSrc: [
        "'self'",
        'data:',          // Base64画像
        'https:',         // 外部画像（avatarなど）
      ],
      connectSrc: [
        "'self'",
        API_DOMAIN,
        ...(isDev ? ['ws://localhost:*'] : []), // 開発時のWebSocket（HMR）
      ],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
      baseUri: ["'self'"],
      formAction: ["'self'"],
      upgradeInsecureRequests: isDev ? null : [], // 本番のみHTTPS強制
      reportUri: isDev ? null : '/api/csp-report',
    },
    // 開発時はreport-onlyモードで違反を確認
    reportOnly: isDev,
  },

  // HSTS: HTTPSを1年間強制
  strictTransportSecurity: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true, // HSSTSプリロードリストに登録
  },

  // クリックジャッキング防止
  frameguard: { action: 'deny' },

  // MIMEスニッフィング防止
  noSniff: true,

  // リファラー情報の制御
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },

  // X-Powered-By: Express を削除
  hidePoweredBy: true,

  // Cross-Origin Embedder Policy
  crossOriginEmbedderPolicy: { policy: 'require-corp' },
});
```

```typescript
// CSP違反レポートエンドポイント
app.post('/api/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  const report = req.body['csp-report'];

  logger.warn({
    violatedDirective: report?.['violated-directive'],
    blockedUri: report?.['blocked-uri'],
    documentUri: report?.['document-uri'],
  }, 'CSP violation reported');

  res.status(204).end();
});
```

---

## Nonceを使ったインラインスクリプト許可

```typescript
// インラインスクリプトをnonce付きで安全に許可
app.use((req, res, next) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  res.locals.nonce = nonce;
  next();
});

// Helmetの設定でnonceを使う
scriptSrc: [
  "'self'",
  (req, res) => `'nonce-${(res as any).locals.nonce}'`,
],
```

```html
<!-- Reactアプリのエントリーポイント -->
<script nonce="<%= nonce %>" src="/app.js"></script>
```

---

## まとめ

Claude CodeでセキュリティヘッダーをCLAUDE.mdで設計する：

1. **CLAUDE.md** に必須ヘッダー・CSPホワイトリスト方式・HSTS 1年を明記
2. **Helmet** で主要セキュリティヘッダーを一括設定
3. **CSP report-only** で本番適用前に違反を確認
4. **Nonce** でインラインスクリプトを安全に許可

---

*セキュリティヘッダーの設定漏れを検出するスキルは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
