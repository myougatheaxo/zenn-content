---
title: "Claude CodeでAPI不正利用検知を設計する：異常検知・ボット対策・自動ブロック"
emoji: "🚨"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "redis"]
published: true
published_at: "2026-03-14 06:00"
---

## はじめに

レート制限を超えなくても「異常に均一なリクエスト間隔」「1秒に異なるIPから同じアクション」はBot攻撃のサイン——行動ベースの異常検知をClaude Codeに設計させる。

---

## CLAUDE.mdにAPI不正利用検知設計ルールを書く

```markdown
## API不正利用検知設計ルール

### 検知対象パターン
- Credential Stuffing: 短時間に多数の異なるアカウントへのログイン試行
- Account Takeover: 1アカウントへの異なるIPからのアクセス集中
- Scraping: Userスキャン（連番ID、全商品ページ巡回）
- Enumeration: 200→404→200→404のパターン

### スコアリングベースブロック
- スコア 0-30: 正常
- スコア 31-70: 疑わしい（CAPTCHA要求）
- スコア 71+: 自動ブロック（15分）
- スコア算出: IP, UA, 操作パターン, 時刻帯の複合評価

### アラート基準
- 同一IPから5分で50回以上の認証失敗 → 即ブロック
- 同じアカウントへの異なる10以上のIP → アカウントロック
```

---

## API不正利用検知の生成

```
APIの不正利用（Bot・Credential Stuffing）検知システムを設計してください。

要件：
- 行動ベーススコアリング
- Redis TTLウィンドウ集計
- 段階的ブロック（CAPTCHA→完全ブロック）
- 自動アラート

生成ファイル: src/security/abuseDetection/
```

---

## 生成されるAPI不正利用検知実装

```typescript
// src/security/abuseDetection/scorer.ts — リスクスコアリング

interface RequestContext {
  ip: string;
  userId?: string;
  userAgent: string;
  endpoint: string;
  method: string;
  statusCode: number;
  timestamp: number;
}

interface RiskScore {
  score: number;       // 0-100
  flags: string[];     // 検知したパターン
  action: 'allow' | 'captcha' | 'block';
}

export class AbuseDetector {
  async scoreRequest(ctx: RequestContext): Promise<RiskScore> {
    const flags: string[] = [];
    let score = 0;

    // ===== IP基準チェック =====
    const ipChecks = await Promise.all([
      this.checkIPFailRate(ctx.ip),          // 認証失敗率
      this.checkIPRequestRate(ctx.ip),       // リクエスト頻度
      this.checkIPEndpointDiversity(ctx.ip), // アクセスエンドポイントの多様性
    ]);

    const [failRate, reqRate, endpointDiv] = ipChecks;

    if (failRate.count > 10) {
      score += 30;
      flags.push(`high_fail_rate:${failRate.count}`);
    }
    if (reqRate.rpm > 100) {
      score += 20;
      flags.push(`high_rpm:${reqRate.rpm}`);
    }
    if (endpointDiv.unique > 50) {
      score += 25;
      flags.push(`scraping_pattern:${endpointDiv.unique}_endpoints`);
    }

    // ===== アカウント基準チェック（ログイン時） =====
    if (ctx.userId) {
      const accountIPs = await this.getAccountUniqueIPs(ctx.userId);
      if (accountIPs > 10) {
        score += 40;
        flags.push(`account_multi_ip:${accountIPs}_ips`);
      }
    }

    // ===== User-Agentチェック =====
    const uaRisk = this.analyzeUserAgent(ctx.userAgent);
    score += uaRisk.score;
    if (uaRisk.suspicious) flags.push(`suspicious_ua:${uaRisk.reason}`);

    // ===== タイミングパターンチェック =====
    const timingRisk = await this.analyzeTimingPattern(ctx.ip);
    if (timingRisk.tooRegular) {
      score += 30;
      flags.push('bot_timing_pattern');
    }

    score = Math.min(score, 100);
    const action = score >= 70 ? 'block' : score >= 30 ? 'captcha' : 'allow';

    if (score >= 30) {
      logger.warn({ ip: ctx.ip, userId: ctx.userId, score, flags }, 'Risk score elevated');
    }

    return { score, flags, action };
  }

  // 認証失敗率（5分ウィンドウ）
  private async checkIPFailRate(ip: string): Promise<{ count: number }> {
    const key = `fail:${ip}:${Math.floor(Date.now() / 300_000)}`; // 5分バケット
    const count = parseInt(await redis.get(key) ?? '0');
    return { count };
  }

  // リクエスト頻度（1分）
  private async checkIPRequestRate(ip: string): Promise<{ rpm: number }> {
    const key = `req:${ip}:${Math.floor(Date.now() / 60_000)}`;
    const rpm = parseInt(await redis.get(key) ?? '0');
    return { rpm };
  }

  // エンドポイント多様性（スクレイピング検知）
  private async checkIPEndpointDiversity(ip: string): Promise<{ unique: number }> {
    const key = `endpoints:${ip}:${Math.floor(Date.now() / 3_600_000)}`; // 1時間
    const unique = await redis.scard(key);
    return { unique };
  }

  // アカウントへのユニークIP数
  private async getAccountUniqueIPs(userId: string): Promise<number> {
    const key = `account_ips:${userId}:${Math.floor(Date.now() / 3_600_000)}`;
    return redis.scard(key);
  }

  private analyzeUserAgent(ua: string): { score: number; suspicious: boolean; reason?: string } {
    // Headless Chromiumのデフォルト特徴
    if (/HeadlessChrome/.test(ua)) return { score: 50, suspicious: true, reason: 'headless_chrome' };
    if (!ua || ua.length < 20) return { score: 30, suspicious: true, reason: 'short_ua' };
    // 既知のBotライブラリ
    if (/python-requests|axios\/|Go-http|java\//.test(ua)) {
      return { score: 20, suspicious: true, reason: 'known_bot_ua' };
    }
    return { score: 0, suspicious: false };
  }

  // リクエスト間隔の均一性チェック（Bot特徴: 等間隔リクエスト）
  private async analyzeTimingPattern(ip: string): Promise<{ tooRegular: boolean }> {
    const key = `timing:${ip}`;
    const timestamps = await redis.lrange(key, 0, 9); // 直近10回

    if (timestamps.length < 5) return { tooRegular: false };

    const intervals = timestamps.slice(1).map((t, i) =>
      parseInt(t) - parseInt(timestamps[i])
    );
    const mean = intervals.reduce((a, b) => a + b, 0) / intervals.length;
    const variance = intervals.reduce((sum, i) => sum + Math.pow(i - mean, 2), 0) / intervals.length;
    const cv = Math.sqrt(variance) / mean; // 変動係数

    return { tooRegular: cv < 0.1 }; // 10%未満のバラつきはBot疑い
  }
}
```

```typescript
// src/security/abuseDetection/middleware.ts — Express統合

export function abuseDetectionMiddleware() {
  const detector = new AbuseDetector();

  return async (req: Request, res: Response, next: NextFunction) => {
    const ctx: RequestContext = {
      ip: req.ip,
      userId: req.user?.id,
      userAgent: req.headers['user-agent'] ?? '',
      endpoint: req.path,
      method: req.method,
      statusCode: 200, // pre-flight
      timestamp: Date.now(),
    };

    // リクエスト記録
    const pipeline = redis.pipeline();
    const minBucket = Math.floor(Date.now() / 60_000);
    const hourBucket = Math.floor(Date.now() / 3_600_000);

    pipeline.incr(`req:${ctx.ip}:${minBucket}`);
    pipeline.expire(`req:${ctx.ip}:${minBucket}`, 120);
    pipeline.sadd(`endpoints:${ctx.ip}:${hourBucket}`, ctx.endpoint);
    pipeline.expire(`endpoints:${ctx.ip}:${hourBucket}`, 7200);
    pipeline.lpush(`timing:${ctx.ip}`, String(Date.now()));
    pipeline.ltrim(`timing:${ctx.ip}`, 0, 19);
    pipeline.expire(`timing:${ctx.ip}`, 3600);

    if (ctx.userId) {
      pipeline.sadd(`account_ips:${ctx.userId}:${hourBucket}`, ctx.ip);
      pipeline.expire(`account_ips:${ctx.userId}:${hourBucket}`, 7200);
    }

    await pipeline.exec();

    // スコアリング（重いエンドポイントのみ）
    if (['POST', 'PUT'].includes(req.method) || req.path.includes('/auth')) {
      const risk = await detector.scoreRequest(ctx);

      if (risk.action === 'block') {
        return res.status(429).json({ error: 'Too many requests', retryAfter: 900 });
      }
      if (risk.action === 'captcha') {
        res.setHeader('X-Captcha-Required', '1');
      }
    }

    next();
  };
}
```

---

## まとめ

Claude CodeでAPI不正利用検知を設計する：

1. **CLAUDE.md** に多段スコアリング（0-100）・スコア30でCAPTCHA・70で自動ブロック・5種パターン検知を明記
2. **変動係数（CV）チェック** でリクエスト間隔の均一性を測定——10%未満のバラつきはBotと判定
3. **アカウント多IPチェック** で同一アカウントに異なる10以上のIPが集中したらATO（Account Takeover）として検知
4. **Redisパイプライン** でリクエストごとの記録（頻度・エンドポイント・タイミング）をオーバーヘッドなく集計

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
