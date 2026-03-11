---
title: "Claude CodeでSSRF攻撃を防ぐ：URLバリデーション・プライベートIP遮断・Allowlist設計"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "owasp"]
published: true
published_at: "2026-03-14 18:00"
---

## はじめに

ユーザーが指定したURLへサーバーがリクエストを送る機能——SSRFはそれを悪用してAWSメタデータ（`169.254.169.254`）や内部サービスにアクセスさせる。Claude CodeにSSRF防御を設計させる。

---

## CLAUDE.mdにSSRF防御設計ルールを書く

```markdown
## SSRF防御設計ルール

### URLバリデーション（必須）
- プライベートIPブロック: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- リンクローカル遮断: 169.254.0.0/16（AWSメタデータ等）
- localhostブロック: 127.0.0.0/8, ::1
- DNS解決後のIPも検証（DNS Rebinding対策）

### Allowlist方式
- 許可するドメインを明示的にリスト化
- URLスキーム: https://のみ許可（http://, file://, ftp://禁止）
- リダイレクト追跡: 最大3回、各リダイレクト先も再検証

### 外部HTTPリクエスト
- タイムアウト: 10秒
- レスポンスサイズ: 最大10MB
- User-Agent: カスタム（素のaxiosのままにしない）
```

---

## SSRF防御実装の生成

```
Webアプリ向けSSRF防御システムを設計してください。

要件：
- プライベートIPブロック
- DNS Rebinding対策
- Allowlistによる外部URLフィルタリング
- ユーザー入力URLの安全な取得

生成ファイル: src/security/ssrf/
```

---

## 生成されるSSRF防御実装

```typescript
// src/security/ssrf/urlValidator.ts

import { isIP, isIPv4, isIPv6 } from 'net';
import { Resolver } from 'dns/promises';

// プライベートIPレンジ（CIDR）
const PRIVATE_RANGES = [
  { network: '10.0.0.0',    bits: 8  },   // Class A private
  { network: '172.16.0.0',  bits: 12 },   // Class B private
  { network: '192.168.0.0', bits: 16 },   // Class C private
  { network: '127.0.0.0',   bits: 8  },   // Loopback
  { network: '169.254.0.0', bits: 16 },   // Link-local (AWS metadata!)
  { network: '0.0.0.0',     bits: 8  },   // 現在のネットワーク
  { network: '100.64.0.0',  bits: 10 },   // Carrier-grade NAT
  { network: '::1',         bits: 128 },  // IPv6 loopback
  { network: 'fc00::',      bits: 7  },   // IPv6 unique local
  { network: 'fe80::',      bits: 10 },   // IPv6 link-local
];

function ipToInt(ip: string): number {
  return ip.split('.').reduce((acc, octet) => (acc << 8) | parseInt(octet), 0) >>> 0;
}

function isPrivateIP(ip: string): boolean {
  if (isIPv4(ip)) {
    const ipInt = ipToInt(ip);
    for (const range of PRIVATE_RANGES.filter(r => !r.network.includes(':'))) {
      const networkInt = ipToInt(range.network);
      const mask = (~0 << (32 - range.bits)) >>> 0;
      if ((ipInt & mask) === (networkInt & mask)) return true;
    }
    return false;
  }

  // IPv6の簡易チェック
  const normalized = ip.toLowerCase();
  return normalized === '::1' ||
    normalized.startsWith('fc') ||
    normalized.startsWith('fd') ||
    normalized.startsWith('fe80');
}

export class SSRFValidator {
  private readonly resolver: Resolver;
  private readonly allowedDomains: Set<string>;

  constructor(allowedDomains: string[] = []) {
    this.resolver = new Resolver();
    this.resolver.setServers(['8.8.8.8', '8.8.4.4']); // 信頼できるDNSサーバー
    this.allowedDomains = new Set(allowedDomains);
  }

  async validateUrl(rawUrl: string): Promise<{ valid: boolean; error?: string; url?: URL }> {
    // 1. URLパース
    let url: URL;
    try {
      url = new URL(rawUrl);
    } catch {
      return { valid: false, error: 'Invalid URL format' };
    }

    // 2. スキームチェック（httpsのみ）
    if (url.protocol !== 'https:') {
      return { valid: false, error: `Scheme not allowed: ${url.protocol}` };
    }

    // 3. Allowlistチェック（設定されている場合）
    if (this.allowedDomains.size > 0) {
      const isAllowed = [...this.allowedDomains].some(domain =>
        url.hostname === domain || url.hostname.endsWith(`.${domain}`)
      );
      if (!isAllowed) {
        return { valid: false, error: `Domain not in allowlist: ${url.hostname}` };
      }
    }

    // 4. ホスト名がIPの場合は直接チェック
    if (isIP(url.hostname) !== 0) {
      if (isPrivateIP(url.hostname)) {
        return { valid: false, error: `Private IP address not allowed: ${url.hostname}` };
      }
      return { valid: true, url };
    }

    // 5. DNS解決してIPを検証（DNS Rebinding対策）
    try {
      const [ipv4Results, ipv6Results] = await Promise.allSettled([
        this.resolver.resolve4(url.hostname),
        this.resolver.resolve6(url.hostname),
      ]);

      const allIPs: string[] = [
        ...(ipv4Results.status === 'fulfilled' ? ipv4Results.value : []),
        ...(ipv6Results.status === 'fulfilled' ? ipv6Results.value : []),
      ];

      if (allIPs.length === 0) {
        return { valid: false, error: 'DNS resolution failed' };
      }

      // 解決されたIPのどれかがプライベートなら拒否
      const privateIP = allIPs.find(ip => isPrivateIP(ip));
      if (privateIP) {
        return {
          valid: false,
          error: `DNS resolves to private IP: ${privateIP} (possible SSRF or DNS rebinding attack)`,
        };
      }
    } catch (e) {
      return { valid: false, error: `DNS lookup failed: ${(e as Error).message}` };
    }

    return { valid: true, url };
  }
}
```

```typescript
// src/security/ssrf/safeFetch.ts — SSRF防御付きHTTPクライアント

export class SafeHTTPClient {
  private readonly validator: SSRFValidator;
  private static readonly MAX_REDIRECTS = 3;
  private static readonly TIMEOUT_MS = 10_000;
  private static readonly MAX_RESPONSE_SIZE = 10 * 1024 * 1024; // 10MB

  constructor(allowedDomains: string[] = []) {
    this.validator = new SSRFValidator(allowedDomains);
  }

  async fetch(rawUrl: string, options: RequestInit = {}): Promise<Response> {
    const validation = await this.validator.validateUrl(rawUrl);
    if (!validation.valid) {
      throw new SSRFError(`SSRF validation failed: ${validation.error}`);
    }

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), SafeHTTPClient.TIMEOUT_MS);

    try {
      const response = await fetch(rawUrl, {
        ...options,
        signal: controller.signal,
        redirect: 'manual', // リダイレクトを手動処理
        headers: {
          ...options.headers,
          'User-Agent': 'MyApp/1.0 (+https://myapp.example.com/bot)',
        },
      });

      // リダイレクト追跡（各宛先も再検証）
      if (response.status >= 300 && response.status < 400) {
        return this.handleRedirect(response, options, SafeHTTPClient.MAX_REDIRECTS);
      }

      // レスポンスサイズ制限
      const contentLength = response.headers.get('content-length');
      if (contentLength && parseInt(contentLength) > SafeHTTPClient.MAX_RESPONSE_SIZE) {
        throw new SSRFError(`Response too large: ${contentLength} bytes`);
      }

      return response;
    } finally {
      clearTimeout(timeout);
    }
  }

  private async handleRedirect(
    response: Response,
    options: RequestInit,
    remainingRedirects: number
  ): Promise<Response> {
    if (remainingRedirects <= 0) {
      throw new SSRFError('Too many redirects');
    }

    const location = response.headers.get('location');
    if (!location) throw new SSRFError('Redirect without Location header');

    // リダイレクト先URLも再検証
    const validation = await this.validator.validateUrl(location);
    if (!validation.valid) {
      throw new SSRFError(`Redirect target failed SSRF validation: ${validation.error}`);
    }

    return this.fetch(location, options);
  }
}

// 使用例: Webhookプレビュー機能
const httpClient = new SafeHTTPClient([
  'api.github.com',
  'hooks.slack.com',
  'api.stripe.com',
]);

export async function fetchWebhookPreview(url: string): Promise<string> {
  const response = await httpClient.fetch(url);
  const text = await response.text();
  return text.slice(0, 10000); // 最大10,000文字
}
```

```typescript
// src/security/ssrf/middleware.ts — Express ミドルウェア統合

export function ssrfProtectionMiddleware(allowedDomains: string[]) {
  const validator = new SSRFValidator(allowedDomains);

  return async (req: Request, res: Response, next: NextFunction) => {
    // リクエストボディ内のURL フィールドを検証
    const urlFields = extractUrlFields(req.body);

    for (const [field, url] of Object.entries(urlFields)) {
      const result = await validator.validateUrl(url);
      if (!result.valid) {
        logger.warn({ field, url, error: result.error, ip: req.ip }, 'SSRF attempt blocked');
        return res.status(400).json({ error: `Invalid URL in field '${field}': ${result.error}` });
      }
    }

    next();
  };
}

// URLフィールドを再帰的に抽出
function extractUrlFields(obj: unknown, prefix = ''): Record<string, string> {
  if (!obj || typeof obj !== 'object') return {};
  const urls: Record<string, string> = {};

  for (const [key, value] of Object.entries(obj as Record<string, unknown>)) {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === 'string' && /^https?:\/\//.test(value)) {
      urls[fullKey] = value;
    } else if (typeof value === 'object') {
      Object.assign(urls, extractUrlFields(value, fullKey));
    }
  }
  return urls;
}
```

---

## まとめ

Claude CodeでSSRF攻撃を防ぐ：

1. **CLAUDE.md** にプライベートIPブロック・link-local遮断・httpsのみ許可・DNS Rebinding対策を明記
2. **DNS解決後のIPも検証** することで、一見正常なドメインが内部IPに解決されるDNS Rebinding攻撃を防止
3. **リダイレクト追跡** で各リダイレクト先URLも再検証——リダイレクトを経由した迂回攻撃を遮断
4. **Allowlist方式** で想定外のドメインへのリクエストを原則禁止し、例外的に許可するドメインを明示

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
