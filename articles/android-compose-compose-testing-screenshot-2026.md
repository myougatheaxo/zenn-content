---
title: "Compose Screenshot Testing完全ガイド — Roborazzi/スナップショットテスト/CI統合"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Compose Screenshot Testing**（Roborazzi、スナップショットテスト、ゴールデンテスト、CI/CD統合）を解説します。

---

## Roborazziセットアップ

```groovy
// build.gradle
plugins {
    id("io.github.takahirom.roborazzi") version "1.7.0"
}

dependencies {
    testImplementation("io.github.takahirom.roborazzi:roborazzi:1.7.0")
    testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.7.0")
    testImplementation("org.robolectric:robolectric:4.12")
    testImplementation("androidx.compose.ui:ui-test-junit4")
}

android {
    testOptions {
        unitTests {
            isIncludeAndroidResources = true
        }
    }
}
```

---

## スクリーンショットテスト

```kotlin
@RunWith(ParameterizedRobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [33])
class ScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun homeScreen_light() {
        composeTestRule.setContent {
            MyAppTheme(darkTheme = false) {
                HomeScreen()
            }
        }
        composeTestRule.onRoot().captureRoboImage("screenshots/home_light.png")
    }

    @Test
    fun homeScreen_dark() {
        composeTestRule.setContent {
            MyAppTheme(darkTheme = true) {
                HomeScreen()
            }
        }
        composeTestRule.onRoot().captureRoboImage("screenshots/home_dark.png")
    }

    @Test
    fun button_states() {
        composeTestRule.setContent {
            Column {
                Button(onClick = {}) { Text("有効") }
                Button(onClick = {}, enabled = false) { Text("無効") }
            }
        }
        composeTestRule.onRoot().captureRoboImage("screenshots/button_states.png")
    }
}
```

---

## コンポーネント単位テスト

```kotlin
class ComponentScreenshotTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun settingsToggle_checked() {
        composeTestRule.setContent {
            SettingsToggle(title = "ダークモード", description = "外観を切替",
                icon = Icons.Default.DarkMode, checked = true, onCheckedChange = {})
        }
        composeTestRule.onRoot().captureRoboImage("screenshots/settings_toggle_on.png")
    }

    @Test
    fun settingsToggle_unchecked() {
        composeTestRule.setContent {
            SettingsToggle(title = "ダークモード", description = "外観を切替",
                icon = Icons.Default.DarkMode, checked = false, onCheckedChange = {})
        }
        composeTestRule.onRoot().captureRoboImage("screenshots/settings_toggle_off.png")
    }
}

// 実行: ./gradlew recordRoborazziDebug  (ゴールデン画像生成)
// 比較: ./gradlew verifyRoborazziDebug  (差分チェック)
```

---

## まとめ

| ツール | 用途 |
|--------|------|
| Roborazzi | スクリーンショットテスト |
| `captureRoboImage` | 画像キャプチャ |
| `recordRoborazzi` | ゴールデン画像生成 |
| `verifyRoborazzi` | 差分検証 |

- Roborazziでエミュレータ不要のスクリーンショットテスト
- `recordRoborazziDebug`でゴールデン画像を生成
- `verifyRoborazziDebug`でCIで差分チェック
- ライト/ダーク両テーマでテストが重要

---

8種類のAndroidアプリテンプレート（テスト設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [Compose TestingRobot](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-robot-2026)
- [Compose Lint](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lint-2026)
