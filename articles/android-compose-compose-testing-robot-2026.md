---
title: "Compose Testing Robotパターン完全ガイド — テスト可読性/Robot DSL/画面操作の抽象化"
emoji: "🤖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Compose Testing Robot**（Robotパターン、テストDSL、画面操作の抽象化、可読性向上）を解説します。

---

## Robotパターンとは

```kotlin
// ❌ 直接操作: 可読性が低い
@Test
fun loginTest_bad() {
    composeTestRule.onNodeWithTag("email_input").performTextInput("test@example.com")
    composeTestRule.onNodeWithTag("password_input").performTextInput("password123")
    composeTestRule.onNodeWithTag("login_button").performClick()
    composeTestRule.onNodeWithText("ホーム").assertIsDisplayed()
}

// ✅ Robotパターン: 意図が明確
@Test
fun loginTest_good() {
    loginRobot {
        enterEmail("test@example.com")
        enterPassword("password123")
        tapLogin()
    } verify {
        homeScreenIsDisplayed()
    }
}
```

---

## Robot実装

```kotlin
class LoginRobot(private val rule: ComposeContentTestRule) {
    fun enterEmail(email: String) = apply {
        rule.onNodeWithTag("email_input").performTextInput(email)
    }

    fun enterPassword(password: String) = apply {
        rule.onNodeWithTag("password_input").performTextInput(password)
    }

    fun tapLogin() = apply {
        rule.onNodeWithTag("login_button").performClick()
        rule.waitForIdle()
    }

    infix fun verify(block: LoginVerification.() -> Unit) {
        LoginVerification(rule).apply(block)
    }
}

class LoginVerification(private val rule: ComposeContentTestRule) {
    fun homeScreenIsDisplayed() {
        rule.onNodeWithText("ホーム").assertIsDisplayed()
    }

    fun errorMessageIsShown(message: String) {
        rule.onNodeWithText(message).assertIsDisplayed()
    }

    fun loginButtonIsEnabled() {
        rule.onNodeWithTag("login_button").assertIsEnabled()
    }
}

// DSLビルダー
fun ComposeContentTestRule.loginRobot(block: LoginRobot.() -> Unit): LoginRobot {
    return LoginRobot(this).apply(block)
}
```

---

## テスト例

```kotlin
@HiltAndroidTest
class LoginScreenTest {
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun validCredentials_navigatesToHome() {
        composeTestRule.loginRobot {
            enterEmail("user@example.com")
            enterPassword("validPass123")
            tapLogin()
        } verify {
            homeScreenIsDisplayed()
        }
    }

    @Test
    fun emptyEmail_showsError() {
        composeTestRule.loginRobot {
            enterPassword("password123")
            tapLogin()
        } verify {
            errorMessageIsShown("メールアドレスを入力してください")
        }
    }

    @Test
    fun invalidPassword_showsError() {
        composeTestRule.loginRobot {
            enterEmail("user@example.com")
            enterPassword("short")
            tapLogin()
        } verify {
            errorMessageIsShown("パスワードは8文字以上必要です")
        }
    }
}
```

---

## まとめ

| 概念 | 用途 |
|------|------|
| Robot | 画面操作の抽象化 |
| Verification | アサーションの抽象化 |
| DSL | テスト記述の簡潔化 |
| `apply` | メソッドチェーン |

- Robotパターンで「何をするか」と「何を検証するか」を分離
- テストが自然言語に近くなり可読性向上
- UIが変わってもRobotクラスだけ修正すればOK
- `infix fun verify`でDSL的な記述が可能

---

8種類のAndroidアプリテンプレート（テスト設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [Compose ScreenshotTesting](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-screenshot-2026)
- [Hilt Testing](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
