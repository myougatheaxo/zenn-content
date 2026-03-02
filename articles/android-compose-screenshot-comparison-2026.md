---
title: "スクリーンショット比較テスト実践ガイド — Roborazzi/Paparazzi/CI連携"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**スクリーンショット比較テスト**（Roborazzi、Paparazzi、ゴールデン画像管理、差分検出、CI/CD統合）を解説します。

---

## Paparazziセットアップ

```kotlin
// build.gradle.kts
plugins {
    id("app.cash.paparazzi") version "1.3.5"
}
```

---

## Paparazziテスト

```kotlin
class ComponentScreenshotTest {
    @get:Rule val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_6,
        theme = "android:Theme.Material3.DayNight"
    )

    @Test
    fun loginButton_default() {
        paparazzi.snapshot {
            Button(onClick = {}) { Text("ログイン") }
        }
    }

    @Test
    fun userCard_withData() {
        paparazzi.snapshot {
            UserCard(User("1", "テストユーザー", "test@example.com"))
        }
    }

    @Test
    fun loginScreen_lightTheme() {
        paparazzi.snapshot {
            AppTheme(themeMode = ThemeMode.LIGHT) {
                LoginScreen()
            }
        }
    }

    @Test
    fun loginScreen_darkTheme() {
        paparazzi.snapshot {
            AppTheme(themeMode = ThemeMode.DARK) {
                LoginScreen()
            }
        }
    }
}
```

---

## Roborazzi（JVMテスト）

```kotlin
// build.gradle.kts
plugins {
    id("io.github.takahirom.roborazzi") version "1.33.0"
}

class RoborazziScreenshotTest {
    @get:Rule val composeRule = createComposeRule()

    @Test
    fun homeScreen() {
        composeRule.setContent {
            AppTheme { HomeScreen() }
        }
        composeRule.onRoot().captureRoboImage("screenshots/home.png")
    }

    @Test
    fun homeScreen_darkMode() {
        composeRule.setContent {
            AppTheme(themeMode = ThemeMode.DARK) { HomeScreen() }
        }
        composeRule.onRoot().captureRoboImage("screenshots/home_dark.png")
    }

    @Test
    fun homeScreen_japanese() {
        composeRule.setContent {
            CompositionLocalProvider(LocalConfiguration provides Configuration().apply { setLocale(Locale.JAPAN) }) {
                AppTheme { HomeScreen() }
            }
        }
        composeRule.onRoot().captureRoboImage("screenshots/home_ja.png")
    }
}
```

---

## CI/CD連携

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
        with: { java-version: '17', distribution: 'temurin' }
      - run: ./gradlew verifyPaparazziDebug
      - if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshot-failures
          path: '**/build/reports/paparazzi/'
```

---

## ゴールデン画像管理

```bash
# ゴールデン画像を記録（初回 or 更新時）
./gradlew recordPaparazziDebug    # Paparazzi
./gradlew recordRoborazziDebug    # Roborazzi

# 検証（CIで実行）
./gradlew verifyPaparazziDebug    # Paparazzi
./gradlew verifyRoborazziDebug    # Roborazzi
```

---

## まとめ

| ツール | 特徴 |
|--------|------|
| Paparazzi | JVMテスト、高速、デバイス不要 |
| Roborazzi | Robolectric連携、Compose対応 |

- `record`でゴールデン画像を保存
- `verify`でPRの差分を検出
- CIで自動実行、差分があればPRにレポート
- ダークモード/多言語の全パターンを網羅

---

8種類のAndroidアプリテンプレート（スクリーンショットテスト設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-screenshot-test-2026)
- [Compose UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-ui-2026)
- [GitHub Actions CI](https://zenn.dev/myougatheaxo/articles/android-compose-github-actions-ci-2026)
