---
title: "Claude CodeのPlanモードを使いこなす — 複雑タスクを安全に実行する"
emoji: "🤖"
type: "tech"
topics:
  - claudecode
  - ai
  - 開発効率
  - tips
  - 設計
published: true
---

## Planモードとは

Claude CodeのPlanモードは、AIが**実際の変更を加える前に計画だけを提示**するモード。

大きな変更を実行する前に「何をどの順番でやるか」を確認できる。

---

## 使い方

```bash
# インタラクティブにPlanモード開始
claude  # → /plan で切り替え

# 非インタラクティブ
claude --plan "新規機能の実装計画を立てて"
```

---

## Planモードが特に有効な場面

### 1. 複数ファイルにまたがる変更

```
「Userモデルにpremiumフラグを追加して。
 DBスキーマ、バリデーション、API、テスト全部」
```

このような変更は影響範囲が広い。Planモードで先に確認:

```
Claude の計画:
1. prisma/schema.prisma - User model に is_premium フィールド追加
2. src/types/user.ts - User型定義更新
3. src/validators/user.ts - バリデーション追加
4. src/controllers/user.ts - APIレスポンスに is_premium 含める
5. test/user.test.ts - テストケース追加
6. npm run db:migrate - マイグレーション実行

この順番で実行してよいですか？
```

---

### 2. リファクタリング

```bash
claude --plan "src/utils/ 内の重複関数を整理して共通化"
```

何をどこに移動するか先に確認してから実行。

---

### 3. 本番環境に影響する操作

```bash
claude --plan "staging DBのデータを本番DBに移行する手順"
```

実行前に手順を書かせることで、リスクを把握できる。

---

## ExitPlanModeの使い方

Planモードで計画を確認したら `ExitPlanMode` で実行フェーズに移行:

```
Plan承認 → ExitPlanMode → 実際の変更実行
```

---

## Planモードと通常モードの使い分け

| 操作 | モード |
|------|--------|
| 1ファイルの軽微な修正 | 通常 |
| 複数ファイルの大きな変更 | Plan → 確認 → 実行 |
| 本番データへの操作 | Plan必須 |
| 新機能の設計 | Plan (計画だけで完結) |
| バグ修正 (影響範囲小) | 通常 |

---

## CLAUDE.mdでPlanモードを自動適用

```markdown
## 実行前確認が必要な操作

以下の操作は必ずPlanモードで計画を提示してから実行:
- 本番DBへのマイグレーション
- 5ファイル以上にまたがる変更
- 認証・セキュリティ関連の変更
- 外部APIの設定変更
```

---

## 計画の品質を上げるプロンプト

```
「〇〇を実装して。Planモードで:
1. 影響するファイルの一覧
2. 変更順序とその理由
3. テスト計画
4. ロールバック手順
を提示してから実行して」
```

---

## まとめ

Planモードは「実行する前に確認する」習慣をAIに持たせるツール。大きな変更や本番影響がある操作では必ず活用することで、取り返しのつかないミスを防げる。


---

この記事の内容はnoteで公開しているClaude Code完全攻略ガイド（全7章）から抜粋したもの。MCPの構築方法、カスタムスキル設計、マルチエージェント構成まで網羅している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
