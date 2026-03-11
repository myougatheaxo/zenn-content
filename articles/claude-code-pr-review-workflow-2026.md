---
title: "Claude CodeでPRレビューを高速化・標準化する実践ガイド"
emoji: "🤖"
type: "tech"
topics:
  - claudecode
  - コードレビュー
  - github
  - チーム開発
  - 開発効率
published: true
---

## PRレビューにかかる時間の現実

エンジニアがPRレビューに使う時間は1日平均1-2時間と言われる。

Claude Codeを使えば:
- **機械的なチェックを自動化** (スタイル、命名規則、基本的なバグ)
- **人間は本質的な判断に集中** (設計、ビジネスロジック、アーキテクチャ)

---

## 基本: PRレビューコマンド

`.claude/commands/review-pr.md`:

```markdown
# PRレビュー

現在のブランチと main の差分をレビュー (`git diff main...HEAD`):

## コードの品質
1. 命名規則の一貫性 (チームの規約に従っているか)
2. 関数の長さ (30行超えは分割を検討)
3. 複雑度 (ネストが3層以上はリファクタ候補)
4. コメント (自明でない箇所に説明があるか)

## バグリスク
1. nullチェック・エラーハンドリングの漏れ
2. 非同期処理の await 忘れ
3. 配列の境界チェック
4. 型の不一致

## セキュリティ
1. ユーザー入力のバリデーション
2. 認証・認可の確認

## テスト
1. テストケースの網羅性
2. エッジケースのカバー

出力: 🔴必須 / 🟡推奨 / 🟢提案 で重要度を付けること
```

---

## GitHubのPRコメントを自動投稿

```python
# tools/review_pr.py
import subprocess, sys
import anthropic

def review_pr(pr_number: int):
    # diffを取得
    diff = subprocess.check_output(
        ['git', 'diff', 'main...HEAD'],
        text=True
    )

    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"以下のdiffをレビューして。問題点を🔴/🟡/🟢で分類:

{diff}"
        }]
    )

    review = response.content[0].text

    # gh CLIでPRにコメント投稿
    subprocess.run([
        'gh', 'pr', 'comment', str(pr_number),
        '--body', f"## Claude Code レビュー

{review}"
    ])

if __name__ == '__main__':
    review_pr(int(sys.argv[1]))
```

---

## 差分の大きなPRを効率的にレビュー

大きなPRは一気にレビューせず、ファイル単位で分割:

```bash
claude "git diff main...HEAD --name-only で変更ファイル一覧を取得して、
以下の順でレビュー:
1. まず全体の変更概要を把握
2. テストファイルを先に読んで変更意図を理解
3. 本体コードをレビュー

重要度の高い問題から順に報告"
```

---

## チームレビュー基準の文書化

`.claude/commands/review-checklist.md`:

```markdown
# チームレビューチェックリスト

このPRが以下を満たすか確認:

## 必須チェック (マージブロッカー)
- [ ] 全テストが通過している
- [ ] 破壊的変更にはマイグレーションが含まれる
- [ ] セキュリティ上の問題がない
- [ ] パフォーマンスの著しい劣化がない

## 推奨チェック
- [ ] コードスタイルが一貫している
- [ ] ドキュメントが更新されている
- [ ] ログメッセージが適切

## コンテキスト確認
- [ ] 変更の目的が明確 (PRの説明に書いてある)
- [ ] 影響範囲が把握できる

問題があれば箇所を特定してコメント
```

---

## レビュー品質の測定

```bash
claude "過去30件のPRレビューコメントを分析して:
1. 最も頻出する問題カテゴリ
2. チームで合意できるルール候補
3. Hooksで自動化できるチェック

git log --oneline -30 で一覧取得してから分析"
```

---

## まとめ

| レビューの種類 | Claude Codeの役割 | 人間の役割 |
|--------------|----------------|----------|
| スタイル・命名 | 自動チェック | 最終確認のみ |
| バグリスク | 検出・報告 | 判断 |
| 設計判断 | 選択肢提示 | 決定 |
| ビジネスロジック | コンテキスト提供 | レビュー |

人間がやるべき「判断」に集中できるように、機械的なチェックはClaude Codeに任せる。これだけでレビュー時間は半分以下になる。


---

この記事の内容はnoteで公開しているClaude Code完全攻略ガイド（全7章）から抜粋したもの。MCPの構築方法、カスタムスキル設計、マルチエージェント構成まで網羅している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
