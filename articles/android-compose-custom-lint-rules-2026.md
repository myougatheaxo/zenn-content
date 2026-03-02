---
title: "カスタムLintルール完全ガイド — Detector/Issue/自動修正"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lint"]
published: true
---

## この記事で学べること

**カスタムLintルール**（Detector実装、Issue定義、QuickFix自動修正、CI統合）を解説します。

---

## Issue定義

```kotlin
class ComposeLintRegistry : IssueRegistry() {
    override val issues = listOf(
        MissingPreviewAnnotationDetector.ISSUE,
        RememberInComposableDetector.ISSUE
    )
    override val api = CURRENT_API
    override val vendor = Vendor(
        vendorName = "MyLintRules",
        identifier = "com.example.lint"
    )
}
```

---

## Detector実装

```kotlin
class MissingPreviewAnnotationDetector : Detector(), SourceCodeScanner {
    companion object {
        val ISSUE = Issue.create(
            id = "MissingPreview",
            briefDescription = "Composable関数に@Previewがありません",
            explanation = "公開Composable関数には@Previewアノテーションを推奨",
            category = Category.CORRECTNESS,
            priority = 6,
            severity = Severity.WARNING,
            implementation = Implementation(
                MissingPreviewAnnotationDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }

    override fun getApplicableUastTypes() = listOf(UMethod::class.java)

    override fun createUastHandler(context: JavaContext) = object : UElementHandler() {
        override fun visitMethod(node: UMethod) {
            val hasComposable = node.hasAnnotation("androidx.compose.runtime.Composable")
            val hasPreview = node.hasAnnotation("androidx.compose.ui.tooling.preview.Preview")
            val isPublic = node.visibility == UastVisibility.PUBLIC

            if (hasComposable && isPublic && !hasPreview) {
                context.report(
                    ISSUE, node, context.getNameLocation(node),
                    "公開Composable関数に@Previewを追加してください"
                )
            }
        }
    }
}
```

---

## QuickFix（自動修正）

```kotlin
class RememberInComposableDetector : Detector(), SourceCodeScanner {
    companion object {
        val ISSUE = Issue.create(
            id = "MissingRemember",
            briefDescription = "remember{}でラップされていないオブジェクト生成",
            explanation = "Composable内のオブジェクト生成はremember{}でラップすべき",
            category = Category.PERFORMANCE,
            priority = 8,
            severity = Severity.ERROR,
            implementation = Implementation(
                RememberInComposableDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }

    override fun createUastHandler(context: JavaContext) = object : UElementHandler() {
        override fun visitCallExpression(node: UCallExpression) {
            // mutableStateOf() が remember{} の外で呼ばれている検出
            if (node.methodName == "mutableStateOf") {
                val parent = node.uastParent
                if (parent !is ULambdaExpression || !isInsideRemember(parent)) {
                    val fix = LintFix.create()
                        .replace()
                        .text(node.asSourceString())
                        .with("remember { ${node.asSourceString()} }")
                        .build()

                    context.report(ISSUE, node, context.getLocation(node),
                        "remember{}でラップしてください", fix)
                }
            }
        }
    }
}
```

---

## build.gradle設定

```kotlin
// lint-rules モジュール（build.gradle.kts）
plugins {
    id("java-library")
    alias(libs.plugins.kotlin.jvm)
}

dependencies {
    compileOnly(libs.lint.api)
    compileOnly(libs.lint.checks)
    testImplementation(libs.lint.tests)
}

// アプリモジュール
dependencies {
    lintChecks(project(":lint-rules"))
}
```

---

## まとめ

| 要素 | 役割 |
|------|------|
| `IssueRegistry` | Issueの登録 |
| `Detector` | コード解析ロジック |
| `Issue` | 警告/エラー定義 |
| `LintFix` | 自動修正 |

- カスタムLintでプロジェクト固有のルールを強制
- `LintFix`で自動修正を提供
- CI/CDで`./gradlew lint`を必ず実行
- Compose特有のパターン（remember漏れ等）を検出

---

8種類のAndroidアプリテンプレート（Lint設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テスト戦略](https://zenn.dev/myougatheaxo/articles/android-compose-testing-robot-pattern-2026)
- [Gradle Convention Plugin](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-convention-plugin-2026)
- [KSP](https://zenn.dev/myougatheaxo/articles/android-compose-ksp-codegen-2026)
