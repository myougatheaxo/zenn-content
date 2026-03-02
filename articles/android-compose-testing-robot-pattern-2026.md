---
title: "Composeテスト Robotパターン完全ガイド — 保守性の高いUIテスト設計"
emoji: "🤖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Robotパターン**（テストの構造化、Page Object的設計、Compose UIテスト、カスタムアサーション、テスト可読性向上）を解説します。

---

## Robotパターンとは

UIテストのアクション（何をするか）とアサーション（何を確認するか）を分離するパターン。テストの可読性と保守性を大幅に向上させます。

---

## Robot定義

```kotlin
class LoginRobot(private val rule: ComposeTestRule) {
    fun enterEmail(email: String) = apply {
        rule.onNodeWithTag("email_field").performTextInput(email)
    }

    fun enterPassword(password: String) = apply {
        rule.onNodeWithTag("password_field").performTextInput(password)
    }

    fun clickLogin() = apply {
        rule.onNodeWithTag("login_button").performClick()
    }

    fun assertErrorShown(message: String) = apply {
        rule.onNodeWithText(message).assertIsDisplayed()
    }

    fun assertLoginSuccess() = apply {
        rule.onNodeWithTag("home_screen").assertIsDisplayed()
    }

    fun assertLoginButtonEnabled() = apply {
        rule.onNodeWithTag("login_button").assertIsEnabled()
    }

    fun assertLoginButtonDisabled() = apply {
        rule.onNodeWithTag("login_button").assertIsNotEnabled()
    }
}
```

---

## テスト実装

```kotlin
@HiltAndroidTest
class LoginScreenTest {
    @get:Rule val composeRule = createComposeRule()

    private fun loginRobot() = LoginRobot(composeRule)

    @Before
    fun setup() {
        composeRule.setContent {
            LoginScreen(viewModel = FakeLoginViewModel())
        }
    }

    @Test
    fun validCredentials_navigatesToHome() {
        loginRobot()
            .enterEmail("test@example.com")
            .enterPassword("password123")
            .clickLogin()
            .assertLoginSuccess()
    }

    @Test
    fun invalidEmail_showsError() {
        loginRobot()
            .enterEmail("invalid")
            .enterPassword("password123")
            .clickLogin()
            .assertErrorShown("メールアドレスの形式が正しくありません")
    }

    @Test
    fun emptyFields_loginButtonDisabled() {
        loginRobot()
            .assertLoginButtonDisabled()
    }
}
```

---

## 複数画面のRobot連携

```kotlin
class HomeRobot(private val rule: ComposeTestRule) {
    fun clickSettings() = SettingsRobot(rule).also {
        rule.onNodeWithTag("settings_button").performClick()
    }

    fun assertUserName(name: String) = apply {
        rule.onNodeWithText(name).assertIsDisplayed()
    }
}

class SettingsRobot(private val rule: ComposeTestRule) {
    fun clickLogout() = LoginRobot(rule).also {
        rule.onNodeWithTag("logout_button").performClick()
    }

    fun toggleDarkMode() = apply {
        rule.onNodeWithTag("dark_mode_switch").performClick()
    }
}

// 画面遷移を含むテスト
@Test
fun logoutFlow() {
    loginRobot()
        .enterEmail("test@example.com")
        .enterPassword("password")
        .clickLogin()
        .assertLoginSuccess()

    HomeRobot(composeRule)
        .assertUserName("テストユーザー")
        .clickSettings()
        .clickLogout()
        .assertErrorShown("")  // LoginScreenに戻る
}
```

---

## LazyColumn用Robot

```kotlin
class ItemListRobot(private val rule: ComposeTestRule) {
    fun scrollToItem(index: Int) = apply {
        rule.onNodeWithTag("item_list").performScrollToIndex(index)
    }

    fun clickItem(title: String) = apply {
        rule.onNodeWithText(title).performClick()
    }

    fun assertItemCount(count: Int) = apply {
        rule.onAllNodesWithTag("list_item").assertCountEquals(count)
    }

    fun swipeToDelete(title: String) = apply {
        rule.onNodeWithText(title).performTouchInput { swipeLeft() }
    }
}
```

---

## まとめ

| 要素 | 役割 |
|------|------|
| Robot | アクション（操作）の定義 |
| `apply` | メソッドチェーン |
| アサーション | 結果検証 |
| 画面遷移 | Robot→Robot返却 |

- Robotパターンでテストの可読性を大幅向上
- `apply`でメソッドチェーン（流れるようなAPI）
- 画面遷移は新Robotを返すことで型安全に
- UI変更時はRobotだけ修正すればテスト本体は不変

---

8種類のAndroidアプリテンプレート（テスト設計パターン適用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-ui-2026)
- [ViewModelテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-viewmodel-2026)
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-screenshot-test-2026)
