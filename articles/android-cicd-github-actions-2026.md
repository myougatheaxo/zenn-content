---
title: "Android CI/CD入門 — GitHub Actionsでビルド・テスト・デプロイを自動化"
emoji: "⚙️"
type: "tech"
topics: ["android", "kotlin", "githubactions", "cicd"]
published: true
---

## この記事で学べること

GitHub Actionsを使えば、push/PRのたびに**ビルド・テスト・リリースを自動実行**できます。Android開発のCI/CDを構築する方法を解説します。

---

## 基本のワークフロー

```yaml
# .github/workflows/android.yml
name: Android CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: Build with Gradle
        run: ./gradlew build

      - name: Run tests
        run: ./gradlew test
```

---

## テスト結果のレポート

```yaml
      - name: Run tests
        run: ./gradlew test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: app/build/reports/tests/
```

テストが失敗しても結果レポートをアップロード。

---

## Lint チェック

```yaml
      - name: Run lint
        run: ./gradlew lint

      - name: Upload lint results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: app/build/reports/lint-results-debug.html
```

---

## 署名付きAABビルド

```yaml
  release:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Decode keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > app/keystore.jks

      - name: Build signed AAB
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: ./gradlew bundleRelease

      - name: Upload AAB
        uses: actions/upload-artifact@v4
        with:
          name: release-aab
          path: app/build/outputs/bundle/release/*.aab
```

---

## GitHub Secrets の設定

| Secret名 | 内容 |
|----------|------|
| `KEYSTORE_BASE64` | `base64 -w 0 keystore.jks` の出力 |
| `KEYSTORE_PASSWORD` | keystoreのパスワード |
| `KEY_ALIAS` | 鍵のエイリアス |
| `KEY_PASSWORD` | 鍵のパスワード |

```bash
# keystoreをBase64エンコード
base64 -w 0 my-release-key.jks > keystore_base64.txt
```

---

## キャッシュで高速化

```yaml
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3
        with:
          cache-read-only: ${{ github.ref != 'refs/heads/main' }}
```

`gradle/actions/setup-gradle`が自動でGradleキャッシュを管理。

---

## PRステータスチェック

```yaml
on:
  pull_request:
    branches: [ main ]

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
      - run: ./gradlew test lint
```

PRにビルド・テスト・lint結果が自動で表示されます。

---

## まとめ

- push/PRで自動ビルド・テスト実行
- GitHub Secretsで署名鍵を安全管理
- Gradle Cacheで2回目以降のビルドを高速化
- テスト・lint結果をアーティファクトとして保存
- AABビルドでGoogle Playデプロイ準備

---

8種類のAndroidアプリテンプレート（CI/CD構成追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アプリ署名完全ガイド](https://zenn.dev/myougatheaxo/articles/android-signing-release-2026)
- [テスト入門（AI時代のAndroidテスト）](https://zenn.dev/myougatheaxo/articles/android-testing-ai-2026)
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
