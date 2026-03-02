---
title: "Compose GitHub Actions完全ガイド — PR自動チェック/テスト自動化/Danger連携"
emoji: "⚙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "githubactions"]
published: true
---

## この記事で学べること

**Compose GitHub Actions**（PRチェック、テスト自動化、コードカバレッジ、リリースワークフロー）を解説します。

---

## PRチェックワークフロー

```yaml
# .github/workflows/pr-check.yml
name: PR Check

on:
  pull_request:
    branches: [ main, develop ]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - uses: gradle/actions/setup-gradle@v3

      - name: Lint
        run: ./gradlew lintDebug

      - name: Unit Tests
        run: ./gradlew testDebugUnitTest

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: '**/build/reports/tests/'

      - name: Upload Lint Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: '**/build/reports/lint-results-debug.html'
```

---

## コードカバレッジ

```yaml
# テストカバレッジ計測+PRコメント
jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run tests with coverage
        run: ./gradlew koverXmlReportDebug

      - name: Add coverage to PR
        uses: mi-kas/kover-report@v1
        with:
          path: app/build/reports/kover/reportDebug.xml
          token: ${{ secrets.GITHUB_TOKEN }}
          title: Code Coverage
          update-comment: true
          min-coverage-overall: 60
          min-coverage-changed-files: 80
```

---

## スクリーンショットテスト

```yaml
jobs:
  screenshot:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Verify screenshots
        run: ./gradlew verifyRoborazziDebug

      - name: Upload diff images
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshot-diffs
          path: '**/build/outputs/roborazzi/'
```

---

## まとめ

| ワークフロー | 用途 |
|-------------|------|
| PR Check | lint+テスト自動実行 |
| Coverage | カバレッジ計測+PR通知 |
| Screenshot | UIの差分検出 |
| Release | AAB生成+Play Store |

- `concurrency`でPR更新時に前のビルドをキャンセル
- `if: always()`でテスト失敗時もレポートをアップロード
- Koverでコードカバレッジを計測
- Roborazziでスクリーンショットの差分チェック

---

8種類のAndroidアプリテンプレート（CI/CD設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose CI/CD](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ci-cd-2026)
- [Compose ScreenshotTesting](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-screenshot-2026)
- [Compose Lint](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lint-2026)
