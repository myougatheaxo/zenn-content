---
title: "Claude Codeで技術的負債を自動検出する - /refactor-suggest スキルの実践"
emoji: "♻️"
type: "tech"
topics: ["claudecode", "リファクタリング", "コードレビュー", "Python", "JavaScript"]
published: true
---

## はじめに

「リファクタリングしたいけど時間がない」「どこから手をつければいいか分からない」

こう感じているエンジニアは多い。技術的負債は放置するほど雪だるま式に膨らむが、日々の機能開発に追われてリファクタリングが後回しになるのは避けられない現実だ。

Claude Codeの `/refactor-suggest` スキルは、このジレンマを解消するために作られた。コードを渡すだけで「どこを」「どう直すべきか」を具体的に提案してくれる。

## /refactor-suggest の基本的な使い方

使い方はシンプルだ。

```bash
/refactor-suggest src/user_service.py
```

ファイルを指定するだけで、Claude Codeがコード全体を解析し、リファクタリング提案をリストアップしてくれる。ディレクトリごと渡すことも可能。

```bash
/refactor-suggest src/services/
```

出力は「問題の種類」「該当箇所」「修正方針」の3点セットで返ってくる。コードを直接書き換えるのではなく、**方針を提示してエンジニアが判断できる形**になっているのがポイント。

## 検出される問題カテゴリ

`/refactor-suggest` が検出する問題は主に4カテゴリに分類される。

**1. DRY違反（コードの重複）**
同じロジックが複数箇所に散らばっている状態を検出する。コピペで増えがちなバリデーション処理や、サービス層に点在する似たようなクエリ構築コードが典型例だ。

**2. 長大関数・複雑すぎるロジック**
1関数が100行を超えていたり、if/elseのネストが深くなっている箇所を指摘する。サイクロマティック複雑度が高い関数は保守コストが跳ね上がるため、分割提案が出る。

**3. デッドコード**
呼ばれていない関数、到達不能なブランチ、使われていないインポートなどを検出する。特に長期運用のプロジェクトでは意外なほど多く見つかる。

**4. 命名の改善提案**
`data`、`temp`、`flag` のような意味の薄い変数名や、動詞が不明確な関数名に対して、意図が伝わる名前を提案してくれる。

## Pythonでの実践例

例として、ユーザー認証サービスの一部を見てみよう。

**リファクタリング前（問題のあるコード）:**

```python
def validate_user_email(email):
    if not email:
        return False
    if '@' not in email:
        return False
    if '.' not in email.split('@')[1]:
        return False
    return True

def validate_admin_email(email):
    if not email:
        return False
    if '@' not in email:
        return False
    if '.' not in email.split('@')[1]:
        return False
    if not email.endswith('.co.jp'):
        return False
    return True

def process_user(name, email, age):
    if not name:
        print("名前が必要です")
        return None
    if not email:
        print("メールが必要です")
        return None
    if not validate_user_email(email):
        print("メール形式が不正です")
        return None
    if age < 0 or age > 150:
        print("年齢が不正です")
        return None
    # ... 以降50行続く処理
    user = {"name": name, "email": email, "age": age}
    return user

def process_admin(name, email, age, department):
    if not name:
        print("名前が必要です")
        return None
    if not email:
        print("メールが必要です")
        return None
    if not validate_admin_email(email):
        print("メール形式が不正です")
        return None
    if age < 0 or age > 150:
        print("年齢が不正です")
        return None
    # ... 以降50行続く処理
    admin = {"name": name, "email": email, "age": age, "dept": department}
    return admin
```

このコードに `/refactor-suggest` を実行すると、以下のような提案が返ってくる。

```
[DRY違反] validate_user_email と validate_admin_email に共通ロジックあり
  → 基底となる is_valid_email(email) を抽出し、admin版は追加ルールのみ実装

[DRY違反] process_user と process_admin のバリデーションブロックが重複
  → _validate_base_fields(name, email, age) として共通化を推奨

[長大関数] process_user が推定80行超え
  → 責務を分割: validate / build_user_dict / save_user の3関数に分ける

[命名] print() でのエラー出力はロギング推奨
  → logging.warning() に置き換えてログレベルを統一すること
```

**リファクタリング後:**

```python
import logging

def is_valid_email(email: str) -> bool:
    if not email or '@' not in email:
        return False
    return '.' in email.split('@')[1]

def is_valid_admin_email(email: str) -> bool:
    return is_valid_email(email) and email.endswith('.co.jp')

def _validate_base_fields(name: str, email: str, age: int, email_validator) -> str | None:
    if not name:
        return "名前が必要です"
    if not email_validator(email):
        return "メール形式が不正です"
    if not (0 <= age <= 150):
        return "年齢が不正です"
    return None

def process_user(name: str, email: str, age: int) -> dict | None:
    error = _validate_base_fields(name, email, age, is_valid_email)
    if error:
        logging.warning(error)
        return None
    return {"name": name, "email": email, "age": age}

def process_admin(name: str, email: str, age: int, department: str) -> dict | None:
    error = _validate_base_fields(name, email, age, is_valid_admin_email)
    if error:
        logging.warning(error)
        return None
    return {"name": name, "email": email, "age": age, "dept": department}
```

行数が半分以下になり、新しいロールを追加するときも `_validate_base_fields` を再利用できるようになった。

## JavaScript/TypeScriptでの実践例

フロントエンドでよく見かけるcallback hell（コールバック地獄）への対応例を見てみよう。

**リファクタリング前（callback hell）:**

```javascript
function loadUserDashboard(userId) {
  fetchUser(userId, function(err, user) {
    if (err) {
      console.error('ユーザー取得失敗', err);
      return;
    }
    fetchOrders(user.id, function(err, orders) {
      if (err) {
        console.error('注文取得失敗', err);
        return;
      }
      fetchRecommendations(user.preferences, function(err, recs) {
        if (err) {
          console.error('レコメンド取得失敗', err);
          return;
        }
        renderDashboard(user, orders, recs);
      });
    });
  });
}
```

`/refactor-suggest` の提案:

```
[コールバック地獄] ネスト深度3以上のコールバックチェーンを検出
  → async/await + Promise.all() へのリファクタリングを推奨

[エラーハンドリング] 各コールバックで個別ハンドリングしているため漏れリスクあり
  → try/catch ブロックで一括ハンドリングに集約すること
```

**リファクタリング後（async/await）:**

```typescript
async function loadUserDashboard(userId: string): Promise<void> {
  try {
    const user = await fetchUser(userId);
    const [orders, recommendations] = await Promise.all([
      fetchOrders(user.id),
      fetchRecommendations(user.preferences),
    ]);
    renderDashboard(user, orders, recommendations);
  } catch (error) {
    console.error('ダッシュボード読み込み失敗:', error);
    showErrorState();
  }
}
```

`Promise.all` による並列実行で、ネットワーク待ち時間も削減できる。

## /code-review との組み合わせワークフロー

`/refactor-suggest` 単体より、`/code-review` と組み合わせることでより効果的なワークフローが組める。

```bash
# Step 1: まず問題の全体像を把握する
/code-review src/legacy.py

# Step 2: リファクタリングの具体的な方針を得る
/refactor-suggest src/legacy.py
```

`/code-review` はバグ・セキュリティ問題・パフォーマンス問題を含む広範なレビューを行う。一方 `/refactor-suggest` はコード構造・保守性に特化している。

**実践的な使い分け:**
- 新規コードのPRレビュー → `/code-review` メイン
- 既存コードの改善計画策定 → `/refactor-suggest` メイン
- レガシーコードの全面改修前 → 両方実行して優先度を決める

## CI/CDでの活用

PRレビュー前に自動でリファクタリング提案を生成するフローを組むことで、レビュアーの負担を減らせる。

```yaml
# .github/workflows/refactor-check.yml
name: Refactor Suggestion

on:
  pull_request:
    branches: [main]

jobs:
  suggest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Get changed files
        id: changed
        run: |
          git diff --name-only origin/main...HEAD \
            | grep -E '\.(py|ts|js)$' \
            | head -10 > changed_files.txt
          cat changed_files.txt

      - name: Run refactor-suggest via Claude Code
        run: |
          while read file; do
            echo "=== $file ===" >> refactor_report.md
            claude -p "/refactor-suggest $file" >> refactor_report.md
          done < changed_files.txt

      - name: Post comment to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('refactor_report.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## リファクタリング提案\n\n${report}`
            });
```

このワークフローを導入すると、PRをオープンした時点で自動的にリファクタリング提案がコメントとして投稿される。レビュアーは提案を参考にしながらコードを確認できる。

## まとめ

`/refactor-suggest` は「いつかリファクタリングしよう」を「今すぐ方針が出る」に変えるスキルだ。

- **DRY違反・長大関数・デッドコード・命名問題**を自動検出
- Python/JavaScript/TypeScript 問わず対応
- `/code-review` と組み合わせることでレビューの質が上がる
- CI/CDに組み込めばPRごとに自動提案が届く

技術的負債の返済に時間をかけすぎる前に、まずスキルに問題を洗い出させてみてほしい。

---

コードレビュー・リファクタリング提案を含むClaude Code用カスタムスキルセットは **Code Review Pack（¥980）** として販売中。

> **[Code Review Pack を見る → PromptWorks](https://prompt-works.jp)**

`/code-review`・`/refactor-suggest`・`/security-check` の3スキルがセットになっており、即日チームの開発フローに導入できる。
