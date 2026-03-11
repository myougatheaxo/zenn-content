---
title: "Claude Codeで依存関係を安全に管理する：脆弱性スキャン・更新戦略・/deps-check"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "security", "npm", "nodejs", "依存関係"]
published: true
---

## はじめに

npm auditで高深刻度の脆弱性を放置したまま開発を続けている——これは珍しいことではない。Claude Codeを使って依存関係管理を体系化し、セキュリティを継続的に維持する方法を紹介する。

---

## CLAUDE.mdに依存関係ルールを書く

```markdown
## 依存関係ルール

### パッケージ追加前チェック（必須）
- npm/PyPIページで週次DL数・最終更新日を確認
- 基準: 1万DL/週以上 + 2年以内に更新あり
- postinstallスクリプトがある場合は内容を必ず確認（マルウェアリスク）
- 公式ドキュメントに記載のないパッケージは追加しない

### バージョン管理
- package.jsonはピン留め（^や~を使わない）: "express": "4.18.2"
- ロックファイルは必ずコミット
- ロックファイルを手動編集しない

### セキュリティ監視
- 依存関係変更後に必ず `npm audit` を実行
- CIでHigh/Critical脆弱性があるとビルド失敗
- 週次でnpm auditのスケジュール実行（GitHub Actions）

### 禁止
- 週次DL数10,000未満のパッケージ（特別な理由がない限り）
- 2年以上更新がないパッケージ
- Critical CVEが未修正のパッケージ
```

---

## npm auditの結果を分析させる

```
npm auditの出力を分析して対応計画を作成してください。

[npm audit --json の出力をここに貼る]

各脆弱性について：
1. 深刻度と影響範囲
2. 修正方法（更新 / ピン留め / パッケージ代替）
3. 修正による破壊的変更のリスク
4. 対応優先度
```

---

## 依存関係更新計画の生成

```
package.jsonの依存関係更新計画を作成してください。

[package.jsonをここに貼る]

各パッケージについて：
1. 現在 vs 最新バージョン
2. 変更タイプ（patch / minor / major）
3. 主な変更点（CHANGELOGから）
4. 推奨更新順序（リスクの低いものから）
```

---

## CIでの自動脆弱性チェック

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    paths:
      - 'package*.json'
      - 'pnpm-lock.yaml'
  schedule:
    - cron: '0 9 * * 1'  # 毎週月曜9時

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 依存関係インストール
        run: npm ci

      - name: npm audit（High/Critical以上で失敗）
        run: npm audit --audit-level=high

      - name: 古いパッケージのチェック
        run: npx npm-check-updates --target minor --format group
```

---

## /deps-checkスキルの出力例

Security Packの `/deps-check` を実行すると：

```
依存関係セキュリティチェック
===========================
高深刻度（即時対応）:
  - lodash@4.17.19 → 4.17.21（プロトタイプ汚染 CVE-2021-23337）
  - vm2@3.9.17 → 削除推奨（メンテナンス終了、RCE脆弱性）

更新推奨（minor, 非破壊的）:
  - express: 4.18.2 → 4.19.2
  - typescript: 5.2.2 → 5.4.5

注意:
  - moment.js → date-fns への移行を検討（バンドルサイズ削減）

対応プラン:
  1. lodash 即時更新（npm audit fix --force）
  2. vm2 削除してvm-browserifyに移行
  3. 他は次スプリントで更新
```

---

## パッケージ選定をClaude Codeに相談する

```
このプロジェクトにメール送信機能を追加したい。
最適なパッケージを教えてください。

要件：
- TypeScript対応
- SendGridとSMTP両方をサポート
- 週次DL数100万以上の実績あり
- テスト環境で実際に送信しないモードがある

上位3つの選択肢を比較してください。
```

---

## まとめ

Claude Codeで依存関係を安全に管理する：

1. **CLAUDE.md** にパッケージ追加基準を明示
2. **npm audit出力を渡す** → 対応計画を自動生成
3. **CIで週次チェック** → High/Critical脆弱性でビルド失敗
4. **/deps-check** → 定期的な包括的レビュー

---

*`/deps-check` スキルは **Security Pack（¥1,480）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
