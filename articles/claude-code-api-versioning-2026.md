---
title: "Claude CodeでAPIバージョニングを設計する：後方互換性を保ちながら進化させる"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "api設計", "後方互換性"]
published: true
---

## はじめに

APIを変更するたびに既存クライアントが壊れる——バージョニング戦略を最初に設計すれば、APIを安全に進化させられる。Claude Codeに設計させる。

---

## CLAUDE.mdにAPIバージョニングルールを書く

```markdown
## APIバージョニングルール

### バージョニング方式
- URLパスバージョニング: /api/v1/users, /api/v2/users
  （シンプル・キャッシュ可能・デバッグしやすい）
- ヘッダーバージョニング: 使わない（URLパスを優先）

### バージョン管理
- 新バージョンの追加: 既存バージョンを変更せず新規ファイルで追加
- 廃止予定: Deprecation-Date ヘッダーで通知（6ヶ月前から）
- バージョンの削除: 最低6ヶ月間の廃止予告期間が必要

### 互換性ルール（同一バージョン内）
- フィールドの削除禁止（optional に変更してから削除）
- フィールド名の変更禁止（新フィールドを追加して旧を deprecated に）
- レスポンスの型変更禁止（string → number 等）
- 追加は可（新フィールド・新エンドポイント）

### サポート期間
- 最新バージョン: フルサポート
- 1世代前: 12ヶ月サポート（バグ修正のみ）
- それ以前: 非サポート（移行ガイドを提供）
```

---

## バージョニングルーターの生成

```
ExpressのAPIバージョニング構造を生成してください。

要件：
- URLパスバージョニング: /api/v1/* / /api/v2/*
- バージョンごとにルーターを分離
- v1はv2の実装を共有（差分のみv2で上書き）
- Deprecation-Date ヘッダーをv1レスポンスに自動付与
- バージョン一覧エンドポイント: GET /api/versions

ディレクトリ構造:
src/routes/
  v1/
    users.ts
    orders.ts
    index.ts
  v2/
    users.ts  ← 変更があるもののみ
    index.ts
  router.ts
```

```typescript
// src/routes/router.ts
import { Router } from 'express';
import { v1Router } from './v1';
import { v2Router } from './v2';

export const apiRouter = Router();

// v1 APIに廃止予告ヘッダーを付与
const v1DeprecationMiddleware = (_req: Request, res: Response, next: NextFunction) => {
  res.setHeader('Deprecation', 'true');
  res.setHeader('Deprecation-Date', '2027-01-01'); // 廃止予定日
  res.setHeader(
    'Sunset', 'Sun, 01 Jan 2027 00:00:00 GMT'
  );
  res.setHeader(
    'Link',
    '<https://api.example.com/api/v2>; rel="successor-version"'
  );
  next();
};

apiRouter.use('/v1', v1DeprecationMiddleware, v1Router);
apiRouter.use('/v2', v2Router);

// バージョン情報エンドポイント
apiRouter.get('/versions', (_req, res) => {
  res.json({
    current: 'v2',
    supported: ['v1', 'v2'],
    deprecated: [{ version: 'v1', sunset: '2027-01-01' }],
  });
});
```

---

## v1からv2への移行設計の生成

```
以下のAPI変更を後方互換に設計してください。

変更内容:
- v1: GET /users → { id, name, email, created_at }
- v2変更: nameをfirst_name + last_nameに分割、snake_caseをcamelCaseに変更

要件：
- v1は変更しない（既存クライアントが壊れない）
- v2で新しいフォーマットを提供
- 内部実装は共有する（DRY）
- v1 → v2移行ガイドをJSDocコメントで追加

生成ファイル:
- src/routes/v1/users.ts（既存）
- src/routes/v2/users.ts（新しいフォーマット）
- src/services/userService.ts（共有ビジネスロジック）
```

```typescript
// src/services/userService.ts（内部表現・共有）
interface UserRecord {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  createdAt: Date;
}

// src/routes/v1/users.ts（v1レスポンス形式）
router.get('/users/:id', async (req, res) => {
  const user = await userService.findById(req.params.id);
  // v1フォーマット（snake_case + name結合）
  res.json({
    id: user.id,
    name: `${user.firstName} ${user.lastName}`,
    email: user.email,
    created_at: user.createdAt.toISOString(),
  });
});

// src/routes/v2/users.ts（v2レスポンス形式）
/**
 * v1からの変更点:
 * - nameをfirst_nameとlast_nameに分割（v1では`name`フィールドに結合していた）
 * - snake_case → camelCase（created_at → createdAt）
 */
router.get('/users/:id', async (req, res) => {
  const user = await userService.findById(req.params.id);
  // v2フォーマット（camelCase + 名前分割）
  res.json({
    id: user.id,
    firstName: user.firstName,
    lastName: user.lastName,
    email: user.email,
    createdAt: user.createdAt.toISOString(),
  });
});
```

---

## 廃止予告の自動検出

```python
# .claude/hooks/check_api_version.py
import json, re, sys

data = json.load(sys.stdin)
content = data.get("tool_input", {}).get("content", "") or ""
fp = data.get("tool_input", {}).get("file_path", "")

if not fp or "routes/v" not in fp:
    sys.exit(0)

# v1ルーターへの変更を検出
if "routes/v1" in fp and any(
    method in content for method in ["router.get", "router.post", "router.put", "router.delete"]
):
    print("[API] v1 APIを変更しようとしています。", file=sys.stderr)
    print("[API] v1は廃止予定です。v2に変更を加えるか、新しいv2エンドポイントを作成してください。", file=sys.stderr)
    sys.exit(1)  # 警告（ブロックではない）

sys.exit(0)
```

---

## まとめ

Claude CodeでAPIバージョニングを設計する：

1. **CLAUDE.md** にバージョニング方式・互換性ルール・サポート期間を明記
2. **URLパスバージョニング** で明確なバージョン区分
3. **Deprecationヘッダー** でクライアントに廃止を通知
4. **内部実装の共有** でv1/v2の差分のみ管理

---

*APIバージョニングの設計問題を自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
