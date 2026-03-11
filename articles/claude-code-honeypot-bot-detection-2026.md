---
title: "Claude Codeでハニーポット型ボット対策を設計する：隠しフィールド・タイミング検出・行動分析"
emoji: "🍯"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "redis"]
published: true
published_at: "2026-03-15 21:00"
---

## はじめに

「reCAPTCHAを入れたのにスパム登録が止まらない」——ハニーポットフィールド・タイミング分析・行動フィンガープリントで人間とボットを区別する軽量なボット対策をClaude Codeに設計させる。

---

## CLAUDE.mdにボット対策設計ルールを書く

```markdown
## ハニーポット型ボット対策設計ルール

### 多層防御
1. ハニーポットフィールド: CSSで非表示の入力欄（ボットが入力→即拒否）
2. タイミング検出: フォーム表示→送信間隔（2秒未満→ボット）
3. フィールド入力順序: タブ順以外の入力→自動化ツールの疑い
4. マウス/キーボードイベント: 一切なし→自動送信の疑い

### スコアリング（0-100）
- 30点未満: 人間として扱い、処理続行
- 30-70点: CAPTCHA表示（フォールバック）
- 70点超: 静かに失敗（ボットへエラー内容を開示しない）

### ログ
- ブロック時はIP・UA・スコアをRedisに記録
- 24時間以内に3回ブロック → IPをブラックリスト
```

---

## ボット対策実装の生成

```
ハニーポット型ボット対策システムを設計してください。

要件：
- サーバーサイドハニーポット検証
- タイミング分析
- 行動スコアリング
- IPブラックリスト管理

生成ファイル: src/security/botDetection/
```

---

## 生成されるボット対策実装

```typescript
// src/security/botDetection/honeypot.ts — ハニーポットトークン生成・検証

import crypto from 'crypto';

const HONEYPOT_SECRET = process.env.HONEYPOT_SECRET!;
const MIN_SUBMIT_MS = 2000; // 2秒未満は即ボット

interface HoneypotToken {
  formId: string;
  issuedAt: number;
  sessionId: string;
}

// フォーム表示時にトークン発行（サーバーサイド）
export function issueHoneypotToken(sessionId: string): {
  token: string;
  fields: { name: string; type: string; style: string }[];
} {
  const formId = crypto.randomBytes(8).toString('hex');
  const payload: HoneypotToken = { formId, issuedAt: Date.now(), sessionId };

  const token = Buffer.from(JSON.stringify(payload)).toString('base64url');
  const sig = crypto.createHmac('sha256', HONEYPOT_SECRET).update(token).digest('hex');

  // ハニーポットフィールド（CSSで完全非表示）
  const fields = [
    // 名前が魅力的でボットが埋めたくなるフィールド
    { name: 'website',    type: 'text',  style: 'position:absolute;left:-9999px;opacity:0;pointer-events:none;' },
    { name: 'phone_confirm', type: 'tel', style: 'display:none;' },
    { name: 'address2',  type: 'text',  style: 'visibility:hidden;height:0;width:0;overflow:hidden;' },
  ];

  return { token: `${token}.${sig}`, fields };
}

// 送信時にトークン + ハニーポットフィールドを検証
export function verifyHoneypotToken(
  rawToken: string,
  sessionId: string,
  honeypotValues: Record<string, string>
): { score: number; reasons: string[] } {
  const reasons: string[] = [];
  let score = 0;

  // ハニーポットフィールドが空でない → ボット確定
  const filledHoneypots = Object.entries(honeypotValues).filter(([, v]) => v.trim() !== '');
  if (filledHoneypots.length > 0) {
    score += 100;
    reasons.push(`Honeypot fields filled: ${filledHoneypots.map(([k]) => k).join(', ')}`);
    return { score, reasons }; // 確定スコアなのですぐ返す
  }

  // トークン検証
  const [tokenPart, sig] = rawToken.split('.');
  const expectedSig = crypto.createHmac('sha256', HONEYPOT_SECRET).update(tokenPart).digest('hex');

  if (sig !== expectedSig) {
    score += 80;
    reasons.push('Invalid token signature');
    return { score, reasons };
  }

  const payload: HoneypotToken = JSON.parse(Buffer.from(tokenPart, 'base64url').toString());

  // セッションIDチェック
  if (payload.sessionId !== sessionId) {
    score += 60;
    reasons.push('Session mismatch');
  }

  // タイミング検証
  const elapsedMs = Date.now() - payload.issuedAt;
  if (elapsedMs < MIN_SUBMIT_MS) {
    score += 70;
    reasons.push(`Too fast: ${elapsedMs}ms (min: ${MIN_SUBMIT_MS}ms)`);
  } else if (elapsedMs > 30 * 60 * 1000) { // 30分超は期限切れ
    score += 20;
    reasons.push('Token expired');
  }

  return { score, reasons };
}
```

```typescript
// src/security/botDetection/behaviorAnalyzer.ts — 行動分析

interface BehaviorData {
  mouseEventCount: number;      // マウスイベント数（0=ボットの疑い）
  keystrokeCount: number;       // キー入力数
  focusEventCount: number;      // フォーカスイベント数
  fieldFillOrder: string[];     // フィールド入力順序
  expectedFieldOrder: string[]; // 期待される順序（tabIndex順）
  pasteOnly: boolean;           // クリップボード貼り付けのみ（自動ツールの特徴）
}

export function analyzeBehavior(data: BehaviorData): { score: number; reasons: string[] } {
  const reasons: string[] = [];
  let score = 0;

  // マウスイベント0 → ヘッドレスブラウザの可能性
  if (data.mouseEventCount === 0) {
    score += 30;
    reasons.push('No mouse events detected');
  }

  // キー入力もフォーカスも0 → 直接DOMに書き込み
  if (data.keystrokeCount === 0 && data.focusEventCount === 0) {
    score += 40;
    reasons.push('No keyboard/focus events');
  }

  // 全フィールドをClipboard Pasteのみで入力
  if (data.pasteOnly && data.keystrokeCount < 5) {
    score += 20;
    reasons.push('All fields filled via paste only');
  }

  // フィールド入力順序が期待値と大幅に異なる
  const orderMatch = data.fieldFillOrder.every(
    (field, i) => data.expectedFieldOrder[i] === field
  );
  if (!orderMatch && data.fieldFillOrder.length > 0) {
    score += 15;
    reasons.push('Unexpected field fill order');
  }

  return { score: Math.min(100, score), reasons };
}
```

```typescript
// src/security/botDetection/middleware.ts — Expressミドルウェア

export class BotDetectionMiddleware {
  async check(
    req: Request,
    body: {
      _honeypot_token: string;
      website?: string;
      phone_confirm?: string;
      address2?: string;
      _behavior?: string; // フロントエンドから送られる行動データ（base64 JSON）
    }
  ): Promise<{ allowed: boolean; score: number }> {
    const ip = req.ip!;
    const sessionId = req.session?.id ?? 'anonymous';

    // IPブラックリストチェック
    const blacklisted = await redis.get(`botdetect:blacklist:${ip}`);
    if (blacklisted) {
      logger.info({ ip }, 'Blocked: IP blacklisted');
      return { allowed: false, score: 100 };
    }

    // ハニーポット検証
    const honeypotResult = verifyHoneypotToken(
      body._honeypot_token,
      sessionId,
      { website: body.website ?? '', phone_confirm: body.phone_confirm ?? '', address2: body.address2 ?? '' }
    );

    let totalScore = honeypotResult.score;
    const reasons = [...honeypotResult.reasons];

    // 行動分析（オプション）
    if (body._behavior) {
      const behaviorData: BehaviorData = JSON.parse(Buffer.from(body._behavior, 'base64url').toString());
      const behaviorResult = analyzeBehavior(behaviorData);
      totalScore = Math.min(100, totalScore + behaviorResult.score);
      reasons.push(...behaviorResult.reasons);
    }

    totalScore = Math.min(100, totalScore);

    // スコアに基づく判定
    if (totalScore >= 70) {
      // ブロック: 記録・ブラックリスト管理
      logger.warn({ ip, score: totalScore, reasons }, 'Bot detected');

      await redis.incr(`botdetect:block_count:${ip}`);
      await redis.expire(`botdetect:block_count:${ip}`, 86400);

      const blockCount = parseInt(await redis.get(`botdetect:block_count:${ip}`) ?? '1');
      if (blockCount >= 3) {
        await redis.set(`botdetect:blacklist:${ip}`, '1', { EX: 86400 }); // 24時間BL
      }

      return { allowed: false, score: totalScore };
    }

    return { allowed: true, score: totalScore };
  }
}

// ミドルウェア使用例
router.post('/api/auth/register', async (req, res) => {
  const detector = new BotDetectionMiddleware();
  const { allowed, score } = await detector.check(req, req.body);

  if (!allowed) {
    // ボットへは成功のように見せかけてゴミデータを受け取る（エラー内容を開示しない）
    return res.status(200).json({ message: 'Registration successful' });
    // 実際には処理しない（静かな失敗）
  }

  // 30-70点: CAPTCHAをフロントエンドに要求
  if (score >= 30) {
    return res.status(200).json({ requireCaptcha: true });
  }

  // 30点未満: 通常処理
  await userService.register(req.body);
  res.status(201).json({ success: true });
});
```

---

## まとめ

Claude Codeでハニーポット型ボット対策を設計する：

1. **CLAUDE.md** に3段階防御（ハニーポット→タイミング→行動）・スコアリング閾値・静かな失敗方針を明記
2. **ハニーポットフィールド** は`website`, `phone_confirm`など「ボットが埋めたくなる名前」——CSSで完全非表示にして人間は絶対触れない
3. **タイミング検証** で2秒未満の送信を70点扱い——reCAPTCHAなしで大半の単純ボットを撃退できる
4. **静かな失敗（Silent Fail）** でボットに成功レスポンスを返す——エラー内容を開示しないことでボット側が対策を取りにくくする

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
