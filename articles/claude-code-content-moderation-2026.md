---
title: "Claude Codeでコンテンツモデレーションを設計する：AI審査・スパム検知・人間レビュー"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "ai", "security"]
published: true
---

## はじめに

ユーザー投稿コンテンツの自動審査——AIで有害コンテンツを検知し、疑わしいものは人間レビューキューに送る。段階的モデレーションパイプラインをClaude Codeに設計させる。

---

## CLAUDE.mdにコンテンツモデレーション設計ルールを書く

```markdown
## コンテンツモデレーション設計ルール

### 自動審査ルール
- テキスト: AI分類 + 正規表現パターンの二段階チェック
- 画像: AWS Rekognitionでヌード・暴力・スパム検知
- 判定: safe / suspicious / blocked の3段階

### 人間レビューキュー
- suspicious は人間がレビュー（24時間SLA）
- blockedは即時非表示（自動）
- フォールス・ポジティブ率 < 0.1%を目標

### 過去投稿への遡及
- ユーザーがbanned → 過去投稿を再審査
- 大規模違反は自動で関連アカウントを調査
```

---

## モデレーションシステムの生成

```
コンテンツモデレーションパイプラインを設計してください。

要件：
- テキストコンテンツのAI審査
- 画像審査（AWS Rekognition）
- 人間レビューキュー
- ユーザー報告処理
- モデレーション統計

生成ファイル: src/moderation/
```

---

## 生成されるモデレーション実装

```typescript
// src/moderation/textModerator.ts

type ModerationResult = {
  decision: 'safe' | 'suspicious' | 'blocked';
  categories: string[];
  confidence: number;
  reason?: string;
};

// テキストモデレーション（Claude AI + パターン検知の二段階）
export async function moderateText(content: string, context: {
  userId: string;
  contentType: 'post' | 'comment' | 'profile';
}): Promise<ModerationResult> {

  // Step 1: 高速パターンマッチング（レイテンシ最小）
  const patternResult = checkPatterns(content);
  if (patternResult.decision === 'blocked') {
    return patternResult; // パターンで即ブロック
  }

  // Step 2: AI分類（パターンを通過したもののみ）
  const aiResult = await classifyWithAI(content, context);

  // スコアが高い方を採用
  if (aiResult.confidence > patternResult.confidence) {
    return aiResult;
  }

  return patternResult;
}

// パターンベースのチェック（NGワード・スパムパターン）
function checkPatterns(content: string): ModerationResult {
  const lowerContent = content.toLowerCase();

  // NGワードリスト（DBから取得・キャッシュ）
  for (const pattern of cachedNGPatterns) {
    if (pattern.regex.test(content)) {
      return {
        decision: 'blocked',
        categories: [pattern.category],
        confidence: 1.0,
        reason: `Pattern match: ${pattern.id}`,
      };
    }
  }

  // スパムパターン: 同じURLが3回以上
  const urlMatches = content.match(/https?:\/\/[^\s]+/g) ?? [];
  const uniqueUrls = new Set(urlMatches);
  if (urlMatches.length > uniqueUrls.size * 2 || urlMatches.length > 5) {
    return {
      decision: 'suspicious',
      categories: ['spam'],
      confidence: 0.8,
      reason: 'Repeated URLs detected',
    };
  }

  return { decision: 'safe', categories: [], confidence: 0.9 };
}

// Claude AIによる分類
async function classifyWithAI(content: string, context: {
  userId: string;
  contentType: string;
}): Promise<ModerationResult> {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5-20251001', // 高速・低コストモデルで処理
    max_tokens: 200,
    messages: [{
      role: 'user',
      content: `Classify this user-generated content for a social platform. Respond with JSON only.

Content: "${content.slice(0, 500)}"
Context: ${context.contentType}

JSON format:
{
  "decision": "safe" | "suspicious" | "blocked",
  "categories": ["hate_speech" | "spam" | "violence" | "adult" | "misinformation" | "self_harm"],
  "confidence": 0.0-1.0,
  "reason": "brief explanation if not safe"
}`,
    }],
  });

  try {
    const text = response.content[0].type === 'text' ? response.content[0].text : '{}';
    return JSON.parse(text) as ModerationResult;
  } catch {
    return { decision: 'suspicious', categories: ['parse_error'], confidence: 0.5 };
  }
}
```

```typescript
// src/moderation/imageModerator.ts
import { RekognitionClient, DetectModerationLabelsCommand } from '@aws-sdk/client-rekognition';

const rekognition = new RekognitionClient({ region: 'ap-northeast-1' });

// 画像モデレーション（AWS Rekognition）
export async function moderateImage(
  imageKey: string, // S3オブジェクトキー
  bucket: string
): Promise<ModerationResult> {
  const command = new DetectModerationLabelsCommand({
    Image: {
      S3Object: { Bucket: bucket, Name: imageKey },
    },
    MinConfidence: 70, // 70%以上の確信度で検出
  });

  const result = await rekognition.send(command);
  const labels = result.ModerationLabels ?? [];

  // ブロック対象カテゴリ
  const BLOCK_CATEGORIES = ['Explicit Nudity', 'Violence', 'Hate Symbols'];
  // 要確認カテゴリ
  const SUSPICIOUS_CATEGORIES = ['Suggestive', 'Gambling', 'Tobacco'];

  const blockLabels = labels.filter(l =>
    BLOCK_CATEGORIES.some(cat => l.ParentName === cat || l.Name === cat)
  );
  const suspiciousLabels = labels.filter(l =>
    SUSPICIOUS_CATEGORIES.some(cat => l.ParentName === cat || l.Name === cat)
  );

  if (blockLabels.length > 0) {
    return {
      decision: 'blocked',
      categories: blockLabels.map(l => l.Name ?? ''),
      confidence: Math.max(...blockLabels.map(l => (l.Confidence ?? 0) / 100)),
    };
  }

  if (suspiciousLabels.length > 0) {
    return {
      decision: 'suspicious',
      categories: suspiciousLabels.map(l => l.Name ?? ''),
      confidence: Math.max(...suspiciousLabels.map(l => (l.Confidence ?? 0) / 100)),
    };
  }

  return { decision: 'safe', categories: [], confidence: 0.95 };
}
```

```typescript
// src/moderation/pipeline.ts — モデレーションパイプライン統合

export async function moderateContent(contentId: string): Promise<void> {
  const content = await prisma.content.findUniqueOrThrow({ where: { id: contentId } });

  // テキストと画像を並列審査
  const [textResult, imageResult] = await Promise.all([
    content.text ? moderateText(content.text, {
      userId: content.authorId,
      contentType: content.type,
    }) : Promise.resolve<ModerationResult>({ decision: 'safe', categories: [], confidence: 1 }),

    content.imageKey ? moderateImage(content.imageKey, process.env.S3_BUCKET!) :
      Promise.resolve<ModerationResult>({ decision: 'safe', categories: [], confidence: 1 }),
  ]);

  // 厳しい方の判定を採用
  const finalDecision = chooseSeverest(textResult, imageResult);

  // モデレーション結果を保存
  await prisma.contentModeration.create({
    data: {
      contentId,
      decision: finalDecision.decision,
      categories: finalDecision.categories,
      confidence: finalDecision.confidence,
      textDecision: textResult.decision,
      imageDecision: imageResult.decision,
    },
  });

  switch (finalDecision.decision) {
    case 'blocked':
      // 即時非表示
      await prisma.content.update({
        where: { id: contentId },
        data: { status: 'hidden', hiddenReason: 'auto_moderation' },
      });
      await notifyAuthor(content.authorId, 'content_removed', finalDecision.categories);
      break;

    case 'suspicious':
      // 人間レビューキューに追加（24時間SLA）
      await prisma.reviewQueue.create({
        data: {
          contentId,
          priority: finalDecision.confidence > 0.8 ? 'high' : 'normal',
          dueAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24時間以内にレビュー
          categories: finalDecision.categories,
        },
      });
      break;

    case 'safe':
      await prisma.content.update({
        where: { id: contentId },
        data: { status: 'published', moderatedAt: new Date() },
      });
      break;
  }
}

function chooseSeverest(...results: ModerationResult[]): ModerationResult {
  const priority = { blocked: 3, suspicious: 2, safe: 1 };
  return results.reduce((a, b) =>
    priority[a.decision] > priority[b.decision] ? a : b
  );
}
```

---

## まとめ

Claude Codeでコンテンツモデレーションを設計する：

1. **CLAUDE.md** に二段階審査・3段階判定・24時間SLAを明記
2. **パターン → AI の二段階** でNGワードは低レイテンシ、文脈理解はAI
3. **Claude Haiku** で高速・低コストな分類（100トークン以下のJSONレスポンス）
4. **suspicious は人間レビューキュー** に送ってフォールスポジティブを防ぐ

---

*モデレーション設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
