---
title: "Claude Codeで依存関係更新を自動化する：Renovate・自動マージ・セキュリティPR優先"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "devops", "security"]
published: true
published_at: "2026-03-13 09:00"
---

## はじめに

`npm outdated` を毎週手動で見るのは終わり——Renovateで依存関係更新PRを自動生成し、マイナー・パッチは自動マージ、メジャーは人間がレビューするフローを設計する。Claude Codeに生成させる。

---

## CLAUDE.mdに依存関係更新設計ルールを書く

```markdown
## 依存関係自動更新設計ルール

### Renovate更新ポリシー
- patch: 自動マージ（テスト通過時）
- minor: 自動マージ（テスト通過時）
- major: PRのみ（人間レビュー必須）
- セキュリティ: 即座にPR作成（通常の優先度より高）

### グループ化
- 同一スコープ（@types/*）はまとめてPR1本
- 開発依存（devDependencies）は本番と別グループ
- フレームワーク（next.js, react）は個別PR（影響大）

### スケジュール
- 更新チェック: 毎日AM9時（CI負荷を分散）
- 自動マージ: テスト全通過から30分後
```

---

## Renovate設定の生成

```
Renovateによる依存関係自動更新を設計してください。

要件：
- patch/minor自動マージ
- major更新は人間レビュー
- セキュリティ更新優先
- 更新グループ化
- テスト通過後の自動マージ

生成ファイル: renovate.json, .github/workflows/
```

---

## 生成されるRenovate設定

```json
// renovate.json

{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    "security:openssf-scorecard"
  ],
  "timezone": "Asia/Tokyo",
  "schedule": ["after 9am and before 10am on weekdays"],
  "prHourlyLimit": 5,
  "prConcurrentLimit": 10,

  "vulnerabilityAlerts": {
    "labels": ["security"],
    "automerge": false,
    "schedule": ["at any time"],
    "prPriority": 10
  },

  "packageRules": [
    {
      "description": "patch・minor自動マージ（テスト通過時）",
      "matchUpdateTypes": ["patch", "minor"],
      "matchDepTypes": ["dependencies", "devDependencies"],
      "automerge": true,
      "automergeType": "pr",
      "automergeStrategy": "squash",
      "minimumReleaseAge": "3 days",
      "stabilityDays": 3
    },
    {
      "description": "major更新は人間レビュー必須",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "labels": ["major-update", "needs-review"],
      "reviewers": ["team:backend-leads"]
    },
    {
      "description": "型定義は個別でなくまとめてPR1本",
      "matchPackagePrefixes": ["@types/"],
      "groupName": "TypeScript type definitions",
      "automerge": true
    },
    {
      "description": "Next.js/Reactは個別PR（影響大）",
      "matchPackageNames": ["next", "react", "react-dom"],
      "groupName": null,
      "automerge": false,
      "labels": ["framework-update"]
    },
    {
      "description": "Prisma更新はまとめて（スキーマ影響）",
      "matchPackagePrefixes": ["@prisma/", "prisma"],
      "groupName": "Prisma",
      "automerge": false,
      "labels": ["database"]
    },
    {
      "description": "linter/formatter系はまとめてauto-merge",
      "matchPackageNames": ["eslint", "prettier", "typescript"],
      "matchPackagePrefixes": ["@typescript-eslint/", "eslint-"],
      "groupName": "Linting tools",
      "automerge": true
    },
    {
      "description": "テストフレームワークはまとめてauto-merge",
      "matchPackageNames": ["vitest", "jest", "@testing-library"],
      "matchPackagePrefixes": ["@testing-library/", "@vitest/"],
      "groupName": "Testing tools",
      "automerge": true
    }
  ],

  "postUpdateOptions": ["npmDedupe"],
  "rangeStrategy": "bump",
  "semanticCommits": "enabled"
}
```

```yaml
# .github/workflows/renovate-automerge.yml
# Renovate PRが自動マージ条件を満たしているか確認するワークフロー

name: Auto-merge Renovate PRs

on:
  pull_request:
    types: [opened, synchronize, reopened]
  check_suite:
    types: [completed]

jobs:
  test:
    if: startsWith(github.head_ref, 'renovate/')
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci

      - name: Run tests
        run: npm test

      - name: Run type check
        run: npm run type-check

      - name: Run lint
        run: npm run lint

  automerge:
    needs: test
    if: |
      startsWith(github.head_ref, 'renovate/') &&
      needs.test.result == 'success'
    runs-on: ubuntu-latest

    steps:
      - name: Wait 30 minutes before auto-merge
        run: sleep 1800  # 30分待機（バグ報告の猶予）

      - name: Auto-merge Renovate PR
        uses: pascalgn/automerge-action@v0.16.3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          MERGE_LABELS: "automerge,!major-update,!needs-review"
          MERGE_METHOD: squash
          MERGE_COMMIT_MESSAGE: "pull-request-title"
          UPDATE_LABELS: ""
```

```yaml
# .github/workflows/dependency-review.yml
# PRにサードパーティ依存の脆弱性スキャン

name: Dependency Review

on:
  pull_request:
    paths:
      - 'package*.json'
      - 'yarn.lock'
      - 'pnpm-lock.yaml'

jobs:
  dependency-review:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
          deny-licenses: GPL-3.0, AGPL-3.0  # ライセンスコンフリクト防止
          comment-summary-in-pr: on-failure
```

```typescript
// scripts/check-outdated.ts — 重大な更新の定期レポート

import { execSync } from 'child_process';
import { sendSlackMessage } from '../src/lib/slack';

interface OutdatedPackage {
  current: string;
  wanted: string;
  latest: string;
  location: string;
}

async function generateOutdatedReport(): Promise<void> {
  const output = execSync('npm outdated --json', { encoding: 'utf-8' });
  const outdated: Record<string, OutdatedPackage> = JSON.parse(output || '{}');

  const majorUpdates = Object.entries(outdated).filter(([, info]) => {
    const currentMajor = parseInt(info.current.split('.')[0]);
    const latestMajor = parseInt(info.latest.split('.')[0]);
    return latestMajor > currentMajor;
  });

  if (majorUpdates.length === 0) return;

  const message = [
    `*依存関係メジャーアップデート ${majorUpdates.length}件*`,
    ...majorUpdates.map(([name, info]) =>
      `• \`${name}\`: ${info.current} → ${info.latest}`
    ),
    '',
    'レビューが必要なPRを確認してください: <https://github.com/org/repo/pulls?q=label:major-update|PRリスト>',
  ].join('\n');

  await sendSlackMessage({ channel: '#dependencies', text: message });
}

generateOutdatedReport().catch(console.error);
```

---

## まとめ

Claude Codeで依存関係更新を自動化する：

1. **CLAUDE.md** にpatch/minor自動マージ・major人間レビュー・セキュリティ優先・3日安定化待機を明記
2. **stabilityDays: 3** で公開直後の問題パッケージを避け、コミュニティの評価を待つ
3. **グループ化** で@types/*・linter・testing toolsをまとめてPR本数を削減
4. **dependency-review-action** でmoderate以上の脆弱性導入をPRブロック

---

*依存関係管理のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
