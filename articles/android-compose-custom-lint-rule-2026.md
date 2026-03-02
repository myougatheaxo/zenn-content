---
title: "カスタムLintルール作成ガイド — コード品質自動チェック"
emoji: "🔍"
type: "tech"
topics: ["android", "kotlin", "lint", "codequality"]
published: true
---

## この記事で学べること

**カスタムLintルール**（Detector作成、Issue定義、テスト、Compose用ルール）を解説します。

---

## モジュール構成

```kotlin
// settings.gradle.kts
include(":lint-rules")

// lint-rules/build.gradle.kts
plugins {
    id("java-library")
    id("org.jetbrains.kotlin.jvm")
}

dependencies {
    compileOnly("com.android.tools.lint:lint-api:31.7.3")
    compileOnly("com.android.tools.lint:lint-checks:31.7.3")
    testImplementation("com.android.tools.lint:lint-tests:31.7.3")
    testImplementation("junit:junit:4.13.2")
}

// app/build.gradle.kts
dependencies {
    lintChecks(project(":lint-rules"))
}
```

---

## カスタムDetector

```kotlin
// Composable関数名がPascalCaseか検証
class ComposableNamingDetector : Detector(), SourceCodeScanner {

    override fun getApplicableUastTypes() = listOf(UMethod::class.java)

    override fun createUastHandler(context: JavaContext) =
        object : UElementHandler() {
            override fun visitMethod(node: UMethod) {
                val hasComposable = node.annotations.any {
                    it.qualifiedName == "androidx.compose.runtime.Composable"
                }

                if (hasComposable && node.returnType == PsiTypes.voidType()) {
                    val name = node.name
                    if (name.isNotEmpty() && name[0].isLowerCase()) {
                        context.report(
                            ISSUE,
                            node,
                            context.getNameLocation(node),
                            "Composable関数名はPascalCaseにしてください: `$name`"
                        )
                    }
                }
            }
        }

    companion object {
        val ISSUE = Issue.create(
            id = "ComposableNaming",
            briefDescription = "Composable関数名はPascalCase",
            explanation = "Composable関数はUIコンポーネントなので、PascalCaseで命名してください。",
            category = Category.CORRECTNESS,
            priority = 6,
            severity = Severity.WARNING,
            implementation = Implementation(
                ComposableNamingDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

---

## hardcoded文字列検出

```kotlin
class HardcodedStringDetector : Detector(), SourceCodeScanner {

    override fun getApplicableUastTypes() = listOf(UCallExpression::class.java)

    override fun createUastHandler(context: JavaContext) =
        object : UElementHandler() {
            override fun visitCallExpression(node: UCallExpression) {
                if (node.methodName == "Text" || node.methodName == "Button") {
                    node.valueArguments.forEach { arg ->
                        if (arg is ULiteralExpression && arg.isString) {
                            context.report(
                                ISSUE,
                                arg,
                                context.getLocation(arg),
                                "ハードコードされた文字列はstringResourceを使用してください"
                            )
                        }
                    }
                }
            }
        }

    companion object {
        val ISSUE = Issue.create(
            id = "HardcodedComposeString",
            briefDescription = "Compose内のハードコード文字列",
            explanation = "多言語対応のため、stringResource()を使用してください。",
            category = Category.I18N,
            priority = 5,
            severity = Severity.WARNING,
            implementation = Implementation(
                HardcodedStringDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

---

## IssueRegistry

```kotlin
class MyIssueRegistry : IssueRegistry() {
    override val issues = listOf(
        ComposableNamingDetector.ISSUE,
        HardcodedStringDetector.ISSUE
    )

    override val api = CURRENT_API
}
```

```
// lint-rules/src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry
com.example.lint.MyIssueRegistry
```

---

## テスト

```kotlin
class ComposableNamingDetectorTest {

    @Test
    fun detectsLowerCaseComposable() {
        lint().files(
            kotlin("""
                package test
                import androidx.compose.runtime.Composable

                @Composable
                fun myButton() { } // NG

                @Composable
                fun MyButton() { } // OK
            """).indented()
        )
            .issues(ComposableNamingDetector.ISSUE)
            .run()
            .expect("""
                src/test/test.kt:5: Warning: Composable関数名はPascalCaseにしてください: myButton
                    fun myButton() { }
                        ~~~~~~~~
                0 errors, 1 warnings
            """.trimIndent())
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `Detector` | ルール検出ロジック |
| `Issue` | 警告/エラー定義 |
| `IssueRegistry` | ルール登録 |
| `lint-tests` | ルールテスト |

- `SourceCodeScanner`でKotlin/Javaコード解析
- `XmlScanner`でXMLリソース解析
- `lintChecks`でアプリに適用
- CI/CDで自動チェック可能

---

8種類のAndroidアプリテンプレート（Lintルール設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [GitHub Actions CI/CD](https://zenn.dev/myougatheaxo/articles/android-compose-github-actions-ci-2026)
- [R8/ProGuard](https://zenn.dev/myougatheaxo/articles/android-compose-r8-proguard-2026)
- [Version Catalog](https://zenn.dev/myougatheaxo/articles/android-compose-version-catalog-bom-2026)
