---
title: "スクリーンショットテストガイド — Roborazzi/Paparazzi実装"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

Composeの**スクリーンショットテスト**（Roborazzi、Paparazzi）の導入と実装を解説します。

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

dependencies {
    testImplementation("io.github.takahirom.roborazzi:roborazzi:1.32.2")
    testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.32.2")
    testImplementation("org.robolectric:robolectric:4.14.1")
}
```

---

## 基本のスクリーンショットテスト

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class ScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun homeScreen_default() {
        composeTestRule.setContent {
            AppTheme {
                HomeScreen(
                    uiState = HomeUiState.Success(
                        items = listOf(
                            Item(1, "アイテム1"),
                            Item(2, "アイテム2"),
                            Item(3, "アイテム3")
                        )
                    )
                )
            }
        }

        composeTestRule.onRoot().captureRoboImage(
            "screenshots/home_default.png"
        )
    }

    @Test
    fun homeScreen_loading() {
        composeTestRule.setContent {
            AppTheme {
                HomeScreen(uiState = HomeUiState.Loading)
            }
        }

        composeTestRule.onRoot().captureRoboImage(
            "screenshots/home_loading.png"
        )
    }

    @Test
    fun homeScreen_error() {
        composeTestRule.setContent {
            AppTheme {
                HomeScreen(uiState = HomeUiState.Error("接続エラー"))
            }
        }

        composeTestRule.onRoot().captureRoboImage(
            "screenshots/home_error.png"
        )
    }
}
```

---

## ダークモード対応テスト

```kotlin
@Test
fun homeScreen_darkMode() {
    composeTestRule.setContent {
        AppTheme(darkTheme = true) {
            HomeScreen(
                uiState = HomeUiState.Success(sampleItems)
            )
        }
    }

    composeTestRule.onRoot().captureRoboImage(
        "screenshots/home_dark.png"
    )
}
```

---

## コンポーネント単位のテスト

```kotlin
@Test
fun button_variants() {
    composeTestRule.setContent {
        AppTheme {
            Column(Modifier.padding(16.dp)) {
                Button(onClick = {}) { Text("Primary") }
                Spacer(Modifier.height(8.dp))
                OutlinedButton(onClick = {}) { Text("Outlined") }
                Spacer(Modifier.height(8.dp))
                TextButton(onClick = {}) { Text("Text") }
            }
        }
    }

    composeTestRule.onRoot().captureRoboImage(
        "screenshots/button_variants.png"
    )
}

@Test
fun card_item() {
    composeTestRule.setContent {
        AppTheme {
            ItemCard(
                item = Item(1, "テストアイテム", "説明文がここに入ります")
            )
        }
    }

    composeTestRule.onRoot().captureRoboImage(
        "screenshots/card_item.png"
    )
}
```

---

## Gradleコマンド

```bash
# スクリーンショットの記録（ゴールデン画像の生成）
./gradlew recordRoborazziDebug

# スクリーンショットの検証（差分チェック）
./gradlew verifyRoborazziDebug

# 差分レポートの確認
# build/reports/roborazzi/
```

---

## CI連携

```yaml
# .github/workflows/screenshot.yml
name: Screenshot Test
on: [pull_request]

jobs:
  screenshot:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Verify Screenshots
        run: ./gradlew verifyRoborazziDebug
      - name: Upload Diff
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshot-diff
          path: app/build/reports/roborazzi/
```

---

## まとめ

- Roborazziでスクリーンショットテスト（Robolectric依存）
- `captureRoboImage()`でスナップショット取得
- ライト/ダークモード両方テスト
- 状態ごと（Loading/Success/Error）のスナップショット
- `recordRoborazziDebug`でゴールデン画像生成
- `verifyRoborazziDebug`でCI差分チェック

---

8種類のAndroidアプリテンプレート（テスト設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [UIテスト完全ガイド](https://zenn.dev/myougatheaxo/articles/android-testing-compose-2026)
- [CI/CDガイド](https://zenn.dev/myougatheaxo/articles/android-cicd-github-actions-2026)
- [AIテスト自動化](https://zenn.dev/myougatheaxo/articles/android-testing-ai-2026)
