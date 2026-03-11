---
title: "Claude CodeでContent Security Policyを設計する：XSS防止・nonce・Report-Only移行"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "owasp"]
published: true
published_at: "2026-03-15 17:00"
---

## はじめに

XSSを根本から防ぐ——CSP（Content Security Policy）ヘッダーでインラインスクリプトを禁止し、nonce方式で正規スクリプトのみ許可する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにCSP設計ルールを書く

```markdown
## Content Security Policy設計ルール

### CSPポリシー設計
- default-src: 'none'（ホワイトリスト原則）
- script-src: 'nonce-{random}' のみ（unsafe-inlineは禁止）
- style-src: 'nonce-{random}' + 信頼できるCDN
- img-src: 'self' + data: + 信頼できるCDN
- connect-src: 'self' + 許可する外部API

### デプロイ戦略
1. Content-Security-Policy-Report-Only: レポートのみ（ブロックしない）
2. 2週間レポートを収集して違反を修正
3. Content-Security-Policy: に切り替えてブロック開始

### nonceパターン
- リクエストごとにランダムnonce（16バイトBase64）を生成
- HTMLテンプレートのscriptタグにnonce属性を付与
- CDNキャッシュ: nonceはキャッシュしない設定（Cache-Control: no-store）
```

---

## CSP実装の生成

```
Content Security PolicyによるXSS対策を設計してください。

要件：
- リクエストごとのnonce生成
- Express helmet統合
- CSP違反レポート収集
- Report-Only→Enforce移行フロー

生成ファイル: src/security/csp/
```

---

## 生成されるCSP実装

```typescript
// src/security/csp/middleware.ts — CSPミドルウェア

import crypto from 'crypto';
import helmet from 'helmet';

// リクエストごとのnonceを生成・提供するミドルウェア
export function nonceMiddleware() {
  return (req: Request, res: Response, next: NextFunction) => {
    // 16バイトのランダムnonce
    res.locals.nonce = crypto.randomBytes(16).toString('base64');
    next();
  };
}

// Helmetを使ったCSPヘッダー設定
export function cspMiddleware(options: { reportOnly?: boolean } = {}) {
  return (req: Request, res: Response, next: NextFunction) => {
    const nonce = res.locals.nonce;

    const directives = {
      defaultSrc: ["'none'"],
      scriptSrc: [
        `'nonce-${nonce}'`,     // nonceがあれば実行OK
        "'strict-dynamic'",      // nonceで許可したスクリプトが動的に追加するスクリプトを許可
        'https:',                // フォールバック（古いブラウザ向け）
      ],
      styleSrc: [
        `'nonce-${nonce}'`,
        'https://fonts.googleapis.com',
      ],
      fontSrc: [
        "'self'",
        'https://fonts.gstatic.com',
      ],
      imgSrc: [
        "'self'",
        'data:',                 // Base64画像
        'https://cdn.example.com',
        'https://storage.googleapis.com',
      ],
      connectSrc: [
        "'self'",
        'https://api.example.com',
        ...(process.env.NODE_ENV === 'development' ? ['ws://localhost:*'] : []),
      ],
      frameSrc: ["'none'"],      // iframe禁止
      objectSrc: ["'none'"],     // Flash等禁止
      baseUri: ["'self'"],       // base tag制限
      formAction: ["'self'"],    // form送信先制限
      upgradeInsecureRequests: [], // HTTP→HTTPSに自動アップグレード
      reportUri: ['/api/csp-report'], // 違反レポート送信先
    };

    if (options.reportOnly) {
      res.setHeader(
        'Content-Security-Policy-Report-Only',
        formatCSP(directives)
      );
    } else {
      res.setHeader(
        'Content-Security-Policy',
        formatCSP(directives)
      );
    }

    next();
  };
}

function formatCSP(directives: Record<string, string[]>): string {
  return Object.entries(directives)
    .filter(([, values]) => values.length > 0)
    .map(([directive, values]) => {
      const camelToKebab = directive.replace(/([A-Z])/g, '-$1').toLowerCase();
      return values.length ? `${camelToKebab} ${values.join(' ')}` : camelToKebab;
    })
    .join('; ');
}
```

```typescript
// src/security/csp/reportCollector.ts — CSP違反レポート収集

interface CSPReport {
  'document-uri': string;
  'violated-directive': string;
  'effective-directive': string;
  'blocked-uri': string;
  'source-file'?: string;
  'line-number'?: number;
  'column-number'?: number;
  'status-code': number;
}

// CSP違反レポートエンドポイント
router.post('/api/csp-report', express.json({ type: 'application/csp-report' }), async (req, res) => {
  const report = req.body['csp-report'] as CSPReport;

  if (!report) return res.status(400).end();

  // ブラウザ拡張機能・開発ツールの誤検知をフィルタ
  const isExtension = report['blocked-uri']?.startsWith('chrome-extension:') ||
                      report['blocked-uri']?.startsWith('moz-extension:');
  const isEval = report['violated-directive']?.includes('eval');

  if (!isExtension && !isEval) {
    logger.warn({
      documentUri: report['document-uri'],
      violatedDirective: report['violated-directive'],
      blockedUri: report['blocked-uri'],
      sourceFile: report['source-file'],
      line: report['line-number'],
    }, 'CSP violation detected');

    // Sentryに送信（重大な違反の場合）
    if (report['effective-directive'] === 'script-src') {
      Sentry.captureMessage('CSP script-src violation', {
        level: 'warning',
        extra: report,
      });
    }

    // 集計（同じ違反パターンをカウント）
    const violationKey = `csp:${report['violated-directive']}:${report['blocked-uri']}`;
    await redis.incr(violationKey);
    await redis.expire(violationKey, 86400);
  }

  res.status(204).end();
});

// 週次違反レポート（移行判断用）
export async function generateCSPViolationReport(): Promise<Record<string, number>> {
  const keys = await redis.keys('csp:*');
  const counts: Record<string, number> = {};

  for (const key of keys) {
    const count = parseInt(await redis.get(key) ?? '0');
    counts[key.replace('csp:', '')] = count;
  }

  return counts;
}
```

```typescript
// src/views/layout.ts — nonceをHTMLテンプレートに渡す

// Express.js + EJS の例
app.get('/', nonceMiddleware(), cspMiddleware(), (req, res) => {
  res.render('index', {
    nonce: res.locals.nonce,
    // ...その他のデータ
  });
});
```

```html
<!-- views/index.ejs — nonce属性付きscript/styleタグ -->
<!DOCTYPE html>
<html>
<head>
  <!-- nonce属性がないとCSPでブロックされる -->
  <style nonce="<%= nonce %>">
    body { margin: 0; }
  </style>

  <!-- 外部スクリプトもnonce必須 -->
  <script nonce="<%= nonce %>" src="/app.js"></script>
</head>
<body>
  <!-- ✅ インラインスクリプトもnonce必須 -->
  <script nonce="<%= nonce %>">
    window.__INITIAL_DATA__ = <%- JSON.stringify(initialData) %>;
  </script>

  <!-- ❌ nonce無しは全てブロック（XSSされても実行不可） -->
  <!-- <script>alert('XSS')</script> → ブロック -->
</body>
</html>
```

---

## まとめ

Claude CodeでCSPを設計する：

1. **CLAUDE.md** にdefault-src none・nonce方式・unsafe-inline禁止・Report-Only→Enforce移行フローを明記
2. **リクエストごとの16バイトnonce** でCSPをバイパスできない——攻撃者がnonceを推測できない
3. **`strict-dynamic`** でnonceで許可したスクリプトが動的に追加するスクリプトも実行可能——SPAフレームワーク対応
4. **Report-Only 2週間収集** で既存の正規スクリプトをAllowlistに追加してからEnforceに切り替え

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
