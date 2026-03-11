---
title: "第5章 セキュリティ — OWASP Top 10対策をClaude Codeで自動化"
---

# 第5章 セキュリティ — OWASP Top 10対策をClaude Codeで自動化

セキュリティは「後でやる」が一番危ない分野だ。Claude Codeを使えば、コードを書く段階でセキュリティチェックを組み込める。

---

## 5-1. 入力バリデーション（A03: インジェクション）

全ての外部入力はバリデーションする:

```bash
claude "src/routes/ 以下の全エンドポイントに
Zodによる入力バリデーションを追加。

バリデーション対象:
- リクエストボディ
- クエリパラメータ
- URLパラメータ（型変換含む）

エラー時は統一されたバリデーションエラーレスポンスを返す"
```

実装例:

```typescript
// src/schemas/user.schema.ts
import { z } from 'zod'

export const createUserSchema = z.object({
  body: z.object({
    email: z.string().email('有効なメールアドレスを入力してください'),
    password: z.string()
      .min(8, 'パスワードは8文字以上')
      .regex(/[A-Z]/, '大文字を含めてください')
      .regex(/[0-9]/, '数字を含めてください'),
    name: z.string().min(1).max(100),
  })
})

// ミドルウェア
export const validate = (schema: z.AnyZodObject) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req)
    if (!result.success) {
      throw new AppError(
        'VALIDATION_ERROR',
        'リクエストが不正です',
        400,
        result.error.issues
      )
    }
    next()
  }
}
```

---

## 5-2. SQLインジェクション対策（ORM活用）

Prismaを使う場合は基本的に安全だが、rawクエリに注意:

```bash
claude "src/ 以下でPrismaのrawQueryを使っている箇所を全て検出して。
パラメータが直接埋め込まれているものはプレースホルダーに変更。

$queryRaw のクエリはパラメータの埋め込み方が安全かどうかも確認"
```

危険なパターン:

```typescript
// ❌ SQLインジェクション脆弱
const query = `SELECT * FROM users WHERE email = '${email}'`
await prisma.$queryRaw(query)

// ✅ プレースホルダー使用
await prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`
// または
await prisma.$queryRaw(Prisma.sql`SELECT * FROM users WHERE email = ${email}`)
```

---

## 5-3. 認証バイパス対策（A01: アクセス制御）

```bash
claude "src/routes/ のAPIエンドポイントを分類して:

1. 認証なしでアクセスできるもの（パブリック）
2. 認証が必要なもの（プライベート）
3. 特定のロールが必要なもの（RBAC）

現在の実装で:
- 認証が必要なのに authenticate ミドルウェアがない箇所
- 権限チェックが漏れている箇所
を特定して報告"
```

---

## 5-4. XSS対策（A03）

APIのレスポンスにHTMLが含まれる場合:

```bash
claude "APIレスポンスのXSS対策を実装:

1. Content-Security-Policyヘッダーの設定
2. ユーザー入力のサニタイジング（HTMLを返す場合）
3. JSON APIなのでContent-Type: application/jsonの強制

helmetミドルウェアで対応できる部分とカスタム対応が必要な部分を分けて"
```

---

## 5-5. Hooksでのセキュリティチェック自動化

コードを書くたびに自動チェック:

`.claude/settings.json`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python scripts/security-scan.py ${file}"
          }
        ]
      }
    ]
  }
}
```

`scripts/security-scan.py`:

```python
import sys, re, os

PATTERNS = [
    (r'eval\s*\(', "コードインジェクション: eval()"),
    (r'exec\s*\(', "コマンドインジェクション: exec()"),
    (r'innerHTML\s*=\s*(?![\'"]\s*[\'"])', "XSS: innerHTML代入"),
    (r'password\s*[:=]\s*["\'][^"\']+["\']', "ハードコードパスワード"),
    (r'(?:api[_-]?key|secret[_-]?key|private[_-]?key)\s*[:=]\s*["\'][^"\']{8,}', "ハードコードAPIキー"),
]

if len(sys.argv) < 2 or not os.path.exists(sys.argv[1]):
    sys.exit(0)

with open(sys.argv[1], encoding='utf-8', errors='ignore') as f:
    lines = f.read().split('\n')

issues = []
for i, line in enumerate(lines, 1):
    for pattern, msg in PATTERNS:
        if re.search(pattern, line, re.IGNORECASE):
            issues.append(f"L{i}: {msg}")

if issues:
    print(f"[SECURITY] {sys.argv[1]}:")
    for issue in issues:
        print(f"  {issue}")
```

---

## まとめ

| OWASP Top 10 | 対策 | Claude Codeの活用 |
|-------------|------|----------------|
| A01: アクセス制御 | 認証漏れの検出 | ルート一覧を分析させる |
| A03: インジェクション | Zodバリデーション | 全ルートに追加させる |
| A07: 認証の失敗 | JWT + rate limiting | セキュア実装を明示指定 |
| A09: セキュリティログ | 構造化ログ | ミドルウェアとして実装 |

次の章は、テスト戦略だ。
