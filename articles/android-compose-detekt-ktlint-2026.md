---
title: "Detekt/ktlint完全ガイド — 静的解析/カスタムルール/CI統合"
emoji: "🧹"
type: "tech"
topics: ["android", "kotlin", "detekt", "codequality"]
published: true
---

## この記事で学べること

**Detekt/ktlint**（静的解析設定、カスタムルール、コードフォーマット、CI統合、Compose専用ルール）を解説します。

---

## Detekt設定

```kotlin
// build.gradle.kts (project)
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.6"
}

// build.gradle.kts (app)
plugins {
    id("io.gitlab.arturbosch.detekt")
}

detekt {
    config.setFrom("$rootDir/config/detekt/detekt.yml")
    buildUponDefaultConfig = true
    allRules = false
}

dependencies {
    detektPlugins("io.gitlab.arturbosch.detekt:detekt-formatting:1.23.6")
    detektPlugins("io.nlopez.compose.rules:detekt:0.4.5") // Compose専用
}
```

---

## detekt.yml

```yaml
# config/detekt/detekt.yml
complexity:
  LongMethod:
    threshold: 30
  TooManyFunctions:
    thresholdInFiles: 20
    thresholdInClasses: 15

naming:
  FunctionNaming:
    functionPattern: '[a-zA-Z][a-zA-Z0-9]*'  # Composable は大文字開始OK
  TopLevelPropertyNaming:
    constantPattern: '[A-Z][A-Za-z0-9]*'

style:
  MagicNumber:
    active: true
    ignoreNumbers: ['-1', '0', '1', '2']
    ignoreAnnotated: ['Composable', 'Preview']
  MaxLineLength:
    maxLineLength: 120
  WildcardImport:
    active: true

performance:
  SpreadOperator:
    active: true

# Compose ルール
ComposeRules:
  ComposeModifierMissing:
    active: true
  ComposeModifierNotUsed:
    active: true
  ComposeUnstableCollections:
    active: true
```

---

## ktlint設定

```kotlin
// build.gradle.kts
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.1.0"
}

ktlint {
    version.set("1.2.1")
    android.set(true)
    outputToConsole.set(true)
    reporters {
        reporter(ReporterType.HTML)
        reporter(ReporterType.CHECKSTYLE)
    }
}
```

```
# .editorconfig
[*.{kt,kts}]
ktlint_code_style = android_studio
max_line_length = 120
indent_size = 4
insert_final_newline = true
ktlint_function_naming_ignore_when_annotated_with = Composable
```

---

## CI統合

```yaml
# .github/workflows/code-quality.yml
name: Code Quality
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Detekt
        run: ./gradlew detekt

      - name: ktlint
        run: ./gradlew ktlintCheck

      - name: Android Lint
        run: ./gradlew lint

      - name: Upload Reports
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: lint-reports
          path: |
            **/build/reports/detekt/
            **/build/reports/ktlint/
            **/build/reports/lint-results-*.html
```

---

## pre-commit hook

```bash
#!/bin/bash
# .git/hooks/pre-commit
echo "Running ktlint..."
./gradlew ktlintCheck --daemon

if [ $? -ne 0 ]; then
    echo "ktlint failed. Run './gradlew ktlintFormat' to fix."
    exit 1
fi

echo "Running detekt..."
./gradlew detekt --daemon

if [ $? -ne 0 ]; then
    echo "detekt failed. Please fix the issues."
    exit 1
fi
```

---

## まとめ

| ツール | 役割 |
|--------|------|
| Detekt | コード品質/複雑度分析 |
| ktlint | コードフォーマット |
| Android Lint | Android固有チェック |
| Compose Rules | Compose専用ルール |

- Detektで複雑度・命名・パフォーマンスを静的解析
- ktlintでコードスタイルを統一
- Compose専用ルールでModifier漏れ等を検出
- CI + pre-commit hookで品質ゲートを自動化

---

8種類のAndroidアプリテンプレート（静的解析設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムLintルール](https://zenn.dev/myougatheaxo/articles/android-compose-custom-lint-rules-2026)
- [Gradle Convention Plugin](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-convention-plugin-2026)
- [CI/CD](https://zenn.dev/myougatheaxo/articles/android-compose-ci-cd-github-actions-2026)
