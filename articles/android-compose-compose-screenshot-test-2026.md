---
title: "Screenshot Test完全ガイド — Roborazzi/スナップショット/リグレッション検出"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Screenshot Test**（Roborazzi、スナップショットテスト、ビジュアルリグレッション検出）を解説します。

---

## Roborazziセットアップ

```kotlin
// build.gradle.kts
// testImplementation("io.github.takahirom.roborazzi:roborazzi:1.11.0")
// testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.11.0")
// testImplementation("org.robolectric:robolectric:4.12")

@RunWith(AndroidJUnit4::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [33])
class ScreenshotTest {
    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun loginScreen_snapshot() {
        composeRule.setContent {
            AppTheme {
                LoginScreen(onLogin = { _, _ -> })
            }
        }
        composeRule.onRoot().captureRoboImage("screenshots/login_screen.png")
    }
}
```

---

## コンポーネント単位テスト

```kotlin
@RunWith(AndroidJUnit4::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [33])
class ComponentScreenshotTest {
    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun userCard_default() {
        composeRule.setContent {
            AppTheme {
                UserCard(name = "山田太郎", email = "taro@example.com")
            }
        }
        composeRule.onRoot().captureRoboImage("screenshots/user_card_default.png")
    }

    @Test
    fun userCard_darkMode() {
        composeRule.setContent {
            AppTheme(darkTheme = true) {
                UserCard(name = "山田太郎", email = "taro@example.com")
            }
        }
        composeRule.onRoot().captureRoboImage("screenshots/user_card_dark.png")
    }

    @Test
    fun button_states() {
        composeRule.setContent {
            AppTheme {
                Column(Modifier.padding(16.dp)) {
                    Button(onClick = {}) { Text("有効") }
                    Button(onClick = {}, enabled = false) { Text("無効") }
                }
            }
        }
        composeRule.onRoot().captureRoboImage("screenshots/button_states.png")
    }
}
```

---

## CI連携

```yaml
# .github/workflows/screenshot-test.yml
name: Screenshot Test
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: 17, distribution: temurin }
      - run: ./gradlew verifyRoborazziDebug
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: screenshot-diff
          path: '**/build/outputs/roborazzi/**/*_compare.png'
```

---

## まとめ

| ツール | 用途 |
|-------|------|
| Roborazzi | スクリーンショット撮影 |
| `captureRoboImage` | 画像キャプチャ |
| `verifyRoborazzi` | 差分検出 |
| `recordRoborazzi` | 基準画像更新 |

- Roborazziでスクリーンショットテスト自動化
- ライト/ダークモード両方のスナップショット
- CI/CDでビジュアルリグレッション自動検出
- JVMテストなので高速実行

---

8種類のAndroidアプリテンプレート（テスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ui-test-2026)
- [TestTag](https://zenn.dev/myougatheaxo/articles/android-compose-compose-test-tag-2026)
- [Hilt Testing](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
