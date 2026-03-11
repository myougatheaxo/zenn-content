---
title: "Claude Codeで依存ライブラリの脆弱性を自動検出する - /deps-check スキルの実践"
emoji: "📦"
type: "tech"
topics: ["claudecode", "セキュリティ", "npm", "pip", "CVE"]
published: true
---

## はじめに

`pip install requests` や `npm install axios` を実行するとき、そのライブラリに既知の脆弱性が含まれているかを確認しているだろうか。

大半の開発者は確認していない。それが現実だ。

依存ライブラリは「無意識に取り込む他人のコード」だ。自分で書いたコードより遥かに多い行数が、パッケージとして静かにプロジェクトに混入している。CVE（共通脆弱性識別子）が付いた問題のあるバージョンを使い続けていても、気づかないまま本番稼働しているケースは珍しくない。

Claude Codeの `/deps-check` スキルは、`package.json` や `requirements.txt` をスキャンし、CVEデータベースと照合して既知の脆弱性を検出する。

## /deps-check の仕組み

`/deps-check` は以下の流れで動作する。

1. **依存ファイルの解析**: `package.json`・`requirements.txt`・`Pipfile`・`pyproject.toml` などを読み込み、パッケージ名とバージョンの一覧を抽出する
2. **CVE DBとの照合**: NVD（National Vulnerability Database）およびGHSA（GitHub Security Advisories）の情報と照合し、該当するCVEを特定する
3. **重大度の評価**: CVSSスコアをもとに CRITICAL / HIGH / MEDIUM / LOW の4段階で分類する
4. **修正バージョンの提示**: 脆弱性が修正されたバージョンを特定し、アップグレードコマンドを生成する

## 実際の出力例

**実行コマンド:**

```bash
/deps-check requirements.txt
```

**出力例（Pythonプロジェクト）:**

```
Scanning 48 packages...

[CRITICAL] CVE-2023-32681 - requests==2.28.0
  CVSS: 9.1
  影響: リダイレクト時にAuthorizationヘッダーが意図しないホストに漏洩する
  影響バージョン: requests < 2.31.0
  修正バージョン: requests >= 2.31.0
  修正コマンド: pip install --upgrade requests

[HIGH] CVE-2023-44271 - Pillow==9.5.0
  CVSS: 7.5
  影響: 特定のTIFFファイル解析時にメモリ枯渇が発生するDoS脆弱性
  影響バージョン: Pillow < 10.0.0
  修正バージョン: Pillow >= 10.0.0
  修正コマンド: pip install --upgrade Pillow

[MEDIUM] CVE-2024-0450 - PyYAML==5.4.1
  CVSS: 5.3
  影響: 特定のYAMLドキュメントで任意コードが実行される可能性（yaml.load使用時）
  影響バージョン: PyYAML < 6.0.1
  修正バージョン: PyYAML >= 6.0.1
  修正コマンド: pip install --upgrade PyYAML

Summary: 1 CRITICAL, 1 HIGH, 1 MEDIUM, 0 LOW
一括修正コマンド: pip install --upgrade requests Pillow PyYAML
```

**npm プロジェクトでの実行:**

```bash
/deps-check package.json
```

```
Scanning 124 packages (including transitive dependencies)...

[CRITICAL] CVE-2022-25883 - semver@5.7.1
  CVSS: 9.8
  影響: 正規表現によるReDoS（サービス拒否）。悪意ある入力で無限ループが発生
  影響バージョン: semver < 5.7.2, 6.x < 6.3.1, 7.x < 7.5.2
  修正バージョン: semver >= 7.5.4
  修正コマンド: npm install semver@latest

[HIGH] CVE-2023-26115 - word-wrap@1.2.3
  CVSS: 7.5
  影響: ReDoS。長い入力文字列で正規表現が指数的な時間を消費
  影響バージョン: word-wrap < 1.2.4
  修正バージョン: word-wrap >= 1.2.4
  修正コマンド: npm audit fix

Summary: 1 CRITICAL, 1 HIGH, 0 MEDIUM, 0 LOW
推奨: npm audit fix --force を実行（破壊的変更が含まれる場合は手動確認）
```

CVSSスコア・影響範囲・修正バージョン・実行コマンドまで一気に出力される。

## npm audit / safety との違い

`npm audit` や Python の `safety check` という標準ツールがすでに存在する。`/deps-check` との違いはどこにあるか。

**`npm audit` / `safety` の特徴:**
- ツールのインストールが必要
- 出力がそのままでは読みにくいことがある
- 対処方針は自分で判断する必要がある

**`/deps-check` の強み:**
- Claude Codeのセッション内でそのまま実行できる
- CVEの内容をAIが噛み砕いて説明する（「なぜ危険か」が分かる）
- 修正後の破壊的変更リスクも合わせてコメントされる
- `package.json` と `requirements.txt` を同一セッションで連続実行できる

具体的な違いを見てみよう。

**`npm audit` の生の出力:**

```
# npm audit report

semver  <5.7.2
Severity: critical
Regular Expression Denial of Service in semver - https://github.com/advisories/GHSA-c2qf-rxjj-qqgw
fix available via `npm audit fix`
node_modules/semver
```

**`/deps-check` の出力（同じ脆弱性）:**

```
[CRITICAL] CVE-2022-25883 - semver@5.7.1
  CVSS: 9.8
  影響: バージョン文字列のバリデーション処理に含まれる正規表現が、
        悪意を持って構築された入力に対して指数的な処理時間を要する（ReDoS）。
        ユーザー入力をsemverで検証している箇所があれば、DoS攻撃に利用される可能性あり。
  あなたのコードへの影響: package.json の devDependencies 経由で間接依存
  修正: npm install semver@latest（直接依存の場合）/ npm audit fix（間接依存の場合）
  破壊的変更リスク: 低（semver APIの後方互換性は維持されている）
```

「なぜ危険か」「自分のコードに影響するか」「修正してAPIが壊れないか」まで説明してくれる点が違う。

## CI/CD連携（GitHub Actions例）

PRのたびに自動で依存関係チェックを走らせることで、脆弱性の混入を早期に検知できる。

```yaml
# .github/workflows/deps-check.yml
name: Dependency Security Check

on:
  pull_request:
    paths:
      - 'requirements.txt'
      - 'package.json'
      - 'package-lock.json'
      - 'Pipfile'
      - 'pyproject.toml'
  schedule:
    - cron: '0 9 * * 1'  # 毎週月曜 9:00 JST に定期チェック

jobs:
  deps-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check Python dependencies
        if: hashFiles('requirements.txt') != ''
        run: |
          claude -p "/deps-check requirements.txt" > deps_report.md

      - name: Check Node.js dependencies
        if: hashFiles('package.json') != ''
        run: |
          claude -p "/deps-check package.json" >> deps_report.md

      - name: Post report to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('deps_report.md', 'utf8');
            if (report.includes('[CRITICAL]') || report.includes('[HIGH]')) {
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: `## 依存ライブラリ脆弱性レポート\n\n${report}\n\n> CRITICAL/HIGH が含まれています。マージ前に修正を検討してください。`
              });
              core.setFailed('CRITICAL or HIGH vulnerabilities detected');
            }
```

`paths` フィルタにより、依存ファイルが変更されたPRのみチェックが走る。`schedule` で定期実行も入れておくと、既存依存ライブラリの新規CVE追加にも対応できる。

## まとめ

`/deps-check` は「使っているライブラリに脆弱性がないか」を習慣的に確認するためのスキルだ。

- `package.json`・`requirements.txt` をワンコマンドでスキャン
- CVSSスコア・影響バージョン・修正バージョン・修正コマンドを一括出力
- `npm audit` 等との違いは「AI解説付き」「破壊的変更リスクまでコメント」
- GitHub Actionsに組み込めば依存ファイル変更時に自動チェックが走る

依存ライブラリを増やすたびに `/deps-check` を実行する習慣をつけるだけで、既知の脆弱性を抱えたまま本番リリースするリスクを大幅に下げられる。

---

## Security Packについて

この記事で紹介した `/deps-check` は、**Security Pack**（¥1,480）に含まれています。

- `/security-audit` — OWASP Top 10 自動診断
- `/secret-scanner` — シークレット漏洩検出
- `/deps-check` — 依存ライブラリCVE検出

👉 [PromptWorksで購入する](https://prompt-works.jp)

*みょうが (@myougaTheAxo) — セキュリティ重視のClaudeエンジニア。*
