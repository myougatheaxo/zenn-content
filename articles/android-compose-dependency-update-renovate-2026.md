---
title: "依存関係自動更新完全ガイド — Renovate/Dependabot/Version Catalog"
emoji: "🔃"
type: "tech"
topics: ["android", "kotlin", "gradle", "cicd"]
published: true
---

## この記事で学べること

**依存関係自動更新**（Renovate、Dependabot、Version Catalog連携、セキュリティアップデート、PR自動マージ）を解説します。

---

## Dependabot設定

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
    groups:
      compose:
        patterns:
          - "androidx.compose*"
      firebase:
        patterns:
          - "com.google.firebase*"
    ignore:
      - dependency-name: "com.android.tools.build:gradle"
        update-types: ["version-update:semver-major"]
```

---

## Renovate設定

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchPackagePatterns": ["androidx.compose"],
      "groupName": "Compose",
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchPackagePatterns": ["com.google.firebase"],
      "groupName": "Firebase"
    },
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true
    },
    {
      "matchUpdateTypes": ["major"],
      "labels": ["breaking-change"],
      "automerge": false
    }
  ],
  "vulnerabilityAlerts": {
    "labels": ["security"],
    "automerge": true
  }
}
```

---

## Version Catalog連携

```toml
# gradle/libs.versions.toml
[versions]
compose-bom = "2025.01.01"
kotlin = "2.1.0"
hilt = "2.53.1"
room = "2.6.1"
retrofit = "2.11.0"

[libraries]
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
```

Dependabot/Renovateは `libs.versions.toml` のバージョンを自動更新するPRを作成します。

---

## CI自動テスト付きマージ

```yaml
# .github/workflows/dependency-update.yml
name: Dependency Update Check
on:
  pull_request:
    paths:
      - 'gradle/libs.versions.toml'
      - '**/build.gradle.kts'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew build test lint
      - name: Auto-merge patch updates
        if: contains(github.event.pull_request.labels.*.name, 'dependencies')
        uses: peter-evans/enable-pull-request-automerge@v3
        with:
          merge-method: squash
```

---

## まとめ

| ツール | 特徴 |
|--------|------|
| Dependabot | GitHub標準、設定シンプル |
| Renovate | 高機能、グループ化/自動マージ |
| Version Catalog | バージョン一元管理 |
| CI連携 | テスト通過後に自動マージ |

- `libs.versions.toml`でバージョン一元管理
- Dependabot/Renovateで自動PR作成
- パッチ更新は自動マージ、メジャー更新は手動レビュー
- セキュリティアラートは最優先で自動マージ

---

8種類のAndroidアプリテンプレート（自動更新設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Version Catalog](https://zenn.dev/myougatheaxo/articles/android-compose-version-catalog-bom-2026)
- [GitHub Actions CI](https://zenn.dev/myougatheaxo/articles/android-compose-github-actions-ci-2026)
- [Convention Plugin](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-convention-plugin-2026)
