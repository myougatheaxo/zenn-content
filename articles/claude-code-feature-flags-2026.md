---
title: "Claude Codeでフィーチャーフラグを設計する：安全なデプロイのためのトグル管理"
emoji: "🚦"
type: "tech"
topics: ["claudecode", "featureflags", "nodejs", "typescript", "デプロイ"]
published: true
---

## はじめに

新機能を本番に安全にデプロイしたい——フィーチャーフラグを使うと、コードをデプロイしたまま機能のオン/オフをコントロールできる。Claude Codeにフィーチャーフラグシステムを設計させる。

---

## CLAUDE.mdにフィーチャーフラグルールを書く

```markdown
## フィーチャーフラグルール

### 使用場面（必須）
- 本番環境に影響する新機能は全てフラグで制御
- A/Bテスト対象の機能
- ロールアウト段階（5% → 50% → 100%）

### 実装ルール
- フラグは src/config/flags.ts で一元管理
- フラグ名: snake_case（例: new_checkout_flow, ai_recommendations）
- 各フラグにコメントで: 作成日・担当者・削除予定日を記載
- デフォルト値は必ずfalse（オプトイン型）
- フラグは短命に保つ（最長3ヶ月）→ 期限後は削除

### フラグの種類
1. リリースフラグ: 機能のオン/オフ（コード完成後削除）
2. 実験フラグ: A/Bテスト用（実験終了後削除）
3. オペレーションフラグ: 緊急停止用（長期維持可）

### 禁止
- if (process.env.NODE_ENV === 'development') による分岐（フラグを使う）
- フラグ名にバージョン番号を含める（new_checkout_v2 → new_checkout）
- 90日以上放置されたフラグ（定期的にレビューする）
```

---

## フィーチャーフラグシステムの生成

```
シンプルなフィーチャーフラグシステムを生成してください。

要件：
- 環境変数ベース（Redis不要、スタートアップ向け）
- TypeScript、型安全
- フラグ定義: src/config/flags.ts
- ユーザーIDごとの有効化（一部ユーザーにのみON）
- パーセンテージロールアウト（例: 20%のユーザーに有効化）
- フラグ変更はサーバー再起動不要（設定ファイルを定期ポーリング）

生成するファイル:
- src/config/flags.ts（フラグ定義）
- src/lib/featureFlag.ts（フラグエンジン）
- src/middleware/flagContext.ts（リクエストにフラグコンテキスト付与）
```

---

## 生成されるフラグ定義の例

```typescript
// src/config/flags.ts
export const FLAGS = {
  /**
   * 新チェックアウトフロー
   * 作成: 2026-01-15 @developer1
   * 削除予定: 2026-04-15
   */
  new_checkout_flow: {
    defaultValue: false,
    description: '新しいチェックアウトUI（3ステップ → 1ページ）',
    rolloutPercentage: 20, // 20%のユーザーに有効化
  },

  /**
   * AIレコメンデーション
   * 作成: 2026-02-01 @developer2
   * 削除予定: 2026-05-01
   */
  ai_recommendations: {
    defaultValue: false,
    description: '商品レコメンデーションにAIを使用',
    enabledUserIds: ['user_123', 'user_456'], // 特定ユーザーのみ
  },
} as const;

export type FeatureFlag = keyof typeof FLAGS;
```

```typescript
// src/lib/featureFlag.ts
export class FeatureFlagService {
  isEnabled(flag: FeatureFlag, userId?: string): boolean {
    const config = FLAGS[flag];

    // 特定ユーザーリスト
    if ('enabledUserIds' in config && userId) {
      if (config.enabledUserIds.includes(userId)) return true;
    }

    // パーセンテージロールアウト
    if ('rolloutPercentage' in config && userId) {
      const hash = this.hashUserId(userId);
      return (hash % 100) < config.rolloutPercentage;
    }

    return config.defaultValue;
  }

  private hashUserId(userId: string): number {
    return [...userId].reduce((acc, char) => acc + char.charCodeAt(0), 0);
  }
}

export const featureFlags = new FeatureFlagService();
```

---

## フラグを使ったコード分岐の生成

```
フィーチャーフラグを使ってAPIエンドポイントの新旧ロジックを切り替えてください。

既存: GET /products/:id → 既存のレコメンデーションロジック
新規: ai_recommendations フラグがONの場合 → AIレコメンデーションサービスを呼ぶ

要件：
- フラグOFFはパフォーマンスに影響しない
- フラグON/OFFでレスポンス形式は変えない
- フラグの値はメトリクスとして記録する
```

```typescript
// 生成されるコード例
router.get('/products/:id', async (req, res) => {
  const useAI = featureFlags.isEnabled('ai_recommendations', req.user.id);

  const recommendations = useAI
    ? await aiRecommendationService.get(req.params.id)
    : await legacyRecommendationService.get(req.params.id);

  // フラグの使用状況をメトリクスとして記録
  metrics.increment('feature_flag.used', { flag: 'ai_recommendations', enabled: useAI });

  res.json({ product: await productService.get(req.params.id), recommendations });
});
```

---

## 古いフラグを定期的に検出するHook

```python
# .claude/hooks/check_flag_expiry.py
import json, re, sys
from datetime import datetime

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
fp = data.get("tool_input", {}).get("file_path", "")

if not fp or "flags.ts" not in fp:
    sys.exit(0)

# 削除予定日をコメントから抽出
import re
today = datetime.now().date()
for match in re.finditer(r'削除予定: (\d{4}-\d{2}-\d{2})', content):
    deadline = datetime.strptime(match.group(1), '%Y-%m-%d').date()
    if deadline < today:
        print(f"[FLAG] 期限切れフラグがあります（削除予定: {match.group(1)}）", file=sys.stderr)
        print("[FLAG] 不要になったフラグを削除してください", file=sys.stderr)
        sys.exit(1)  # 警告のみ（ブロックしない）

sys.exit(0)
```

---

## まとめ

Claude Codeでフィーチャーフラグを設計する：

1. **CLAUDE.md** にフラグの使用場面・ライフサイクル・禁止事項を明記
2. **フラグ定義ファイル** に作成日・削除予定日を必ず記載させる
3. **ロールアウト機能** でユーザーIDハッシュによる段階的展開を実装
4. **Claude Code Hooks** で期限切れフラグを自動検出

---

*フィーチャーフラグのコードレビューは **Code Review Pack（¥980）** の `/code-review` で自動化できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
