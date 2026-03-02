---
title: "スクリーンショットテスト完全ガイド — Roborazzi/Paparazzi"
emoji: "📷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**スクリーンショットテスト**（Roborazzi、Paparazzi、視覚的回帰テスト、CI統合）を解説します。

---

## Roborazziセットアップ

```kotlin
// build.gradle.kts (project)
plugins {
    id("io.github.takahirom.roborazzi") version "1.32.2" apply false
}

// build.gradle.kts (app)
plugins {
    id("io.github.takahirom.roborazzi")
}

android {
    testOptions {
        unitTests {
            isIncludeAndroidResources = true
        }
    }
}

dependencies {
    testImplementation("io.github.takahirom.roborazzi:roborazzi:1.32.2")
    testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.32.2")
    testImplementation("org.robolectric:robolectric:4.14.1")
}
```

---

## 基本テスト

```kotlin
@RunWith(ParameterizedRobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [34])
class ScreenshotTest {

    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun homeScreen_default() {
        composeRule.setContent {
            MaterialTheme {
                HomeScreen(
                    uiState = HomeUiState(
                        tasks = listOf(
                            Task("1", "買い物", false),
                            Task("2", "掃除", true),
                            Task("3", "料理", false)
                        )
                    )
                )
            }
        }

        composeRule.onRoot().captureRoboImage(
            "src/test/snapshots/HomeScreen_default.png"
        )
    }

    @Test
    fun homeScreen_empty() {
        composeRule.setContent {
            MaterialTheme {
                HomeScreen(uiState = HomeUiState(tasks = emptyList()))
            }
        }

        composeRule.onRoot().captureRoboImage(
            "src/test/snapshots/HomeScreen_empty.png"
        )
    }

    @Test
    fun homeScreen_loading() {
        composeRule.setContent {
            MaterialTheme {
                HomeScreen(uiState = HomeUiState(isLoading = true))
            }
        }

        composeRule.onRoot().captureRoboImage(
            "src/test/snapshots/HomeScreen_loading.png"
        )
    }
}
```

---

## コンポーネント単位テスト

```kotlin
@Test
fun taskCard_unchecked() {
    composeRule.setContent {
        MaterialTheme {
            TaskCard(
                task = Task("1", "テストタスク", isCompleted = false),
                onToggle = {},
                onDelete = {}
            )
        }
    }

    composeRule
        .onNodeWithText("テストタスク")
        .captureRoboImage("src/test/snapshots/TaskCard_unchecked.png")
}

@Test
fun taskCard_checked() {
    composeRule.setContent {
        MaterialTheme {
            TaskCard(
                task = Task("1", "完了タスク", isCompleted = true),
                onToggle = {},
                onDelete = {}
            )
        }
    }

    composeRule
        .onNodeWithText("完了タスク")
        .captureRoboImage("src/test/snapshots/TaskCard_checked.png")
}
```

---

## ダークモード/多言語テスト

```kotlin
@Test
fun homeScreen_darkMode() {
    composeRule.setContent {
        MaterialTheme(colorScheme = darkColorScheme()) {
            HomeScreen(uiState = sampleUiState)
        }
    }

    composeRule.onRoot().captureRoboImage(
        "src/test/snapshots/HomeScreen_dark.png"
    )
}

@Config(qualifiers = "ja")
@Test
fun homeScreen_japanese() {
    composeRule.setContent {
        MaterialTheme { HomeScreen(uiState = sampleUiState) }
    }
    composeRule.onRoot().captureRoboImage(
        "src/test/snapshots/HomeScreen_ja.png"
    )
}
```

---

## CI統合

```yaml
# .github/workflows/screenshot-test.yml
name: Screenshot Test

on: pull_request

jobs:
  screenshot:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run Screenshot Tests
        run: ./gradlew verifyRoborazziDebug

      - name: Upload Diff
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshot-diff
          path: '**/build/outputs/roborazzi/**/*_compare.png'
```

---

## 実行コマンド

```bash
# スナップショット記録
./gradlew recordRoborazziDebug

# スナップショット検証（差分チェック）
./gradlew verifyRoborazziDebug

# 差分比較画像を出力
./gradlew compareRoborazziDebug
```

---

## まとめ

| ツール | 特徴 |
|--------|------|
| Roborazzi | Robolectric上で高速実行 |
| Paparazzi | エミュレータ不要・高精度 |
| CI統合 | PR時に自動検証 |

- `captureRoboImage`でスナップショット保存
- `verifyRoborazziDebug`で回帰検出
- ダークモード/多言語の網羅的テスト
- CI/CDで自動化してUI品質を保証

---

8種類のAndroidアプリテンプレート（スクリーンショットテスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [統合テスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-integration-2026)
- [ComposeTestRule](https://zenn.dev/myougatheaxo/articles/android-compose-testing-compose-rule-2026)
- [GitHub Actions CI/CD](https://zenn.dev/myougatheaxo/articles/android-compose-github-actions-ci-2026)
