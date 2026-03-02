---
title: "Composeテスト完全ガイド — ComposeTestRule/セマンティクス/スクリーンショット"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Composeテスト**（ComposeTestRule、セマンティクスマッチャー、アクション、アサーション、スクリーンショット）を解説します。

---

## 基本テスト

```kotlin
class LoginScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `ログインボタンが初期状態で無効`() {
        composeTestRule.setContent {
            LoginScreen(onLogin = {})
        }

        composeTestRule
            .onNodeWithText("ログイン")
            .assertIsNotEnabled()
    }

    @Test
    fun `入力後にログインボタンが有効化`() {
        composeTestRule.setContent {
            LoginScreen(onLogin = {})
        }

        composeTestRule
            .onNodeWithText("メールアドレス")
            .performTextInput("test@example.com")

        composeTestRule
            .onNodeWithText("パスワード")
            .performTextInput("password123")

        composeTestRule
            .onNodeWithText("ログイン")
            .assertIsEnabled()
    }
}
```

---

## セマンティクスマッチャー

```kotlin
@Test
fun `各種マッチャーの例`() {
    composeTestRule.setContent { MyScreen() }

    // テキストで検索
    composeTestRule.onNodeWithText("タイトル").assertExists()

    // ContentDescriptionで検索
    composeTestRule.onNodeWithContentDescription("閉じる").performClick()

    // TestTagで検索
    composeTestRule.onNodeWithTag("submit_button").assertIsDisplayed()

    // 複数マッチ
    composeTestRule.onAllNodesWithText("Item").assertCountEquals(5)

    // 階層的検索
    composeTestRule
        .onNodeWithTag("user_list")
        .onChildren()
        .assertCountEquals(3)
}

// TestTag設定
@Composable
fun MyButton() {
    Button(
        onClick = {},
        modifier = Modifier.testTag("submit_button")
    ) { Text("送信") }
}
```

---

## アクション

```kotlin
@Test
fun `スクロールとクリックのテスト`() {
    composeTestRule.setContent { SettingsScreen() }

    // クリック
    composeTestRule.onNodeWithText("通知設定").performClick()

    // テキスト入力
    composeTestRule.onNodeWithTag("search_field").performTextInput("kotlin")

    // スクロール
    composeTestRule.onNodeWithTag("settings_list").performScrollToIndex(10)

    // スワイプ
    composeTestRule.onNodeWithTag("card").performTouchInput {
        swipeLeft()
    }

    // 待機
    composeTestRule.waitUntil(timeoutMillis = 5000) {
        composeTestRule.onAllNodesWithText("完了").fetchSemanticsNodes().isNotEmpty()
    }
}
```

---

## ViewModel連携テスト

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class HomeScreenTest {
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<ComponentActivity>()

    @Test
    fun `データ読み込み後にリストが表示される`() {
        composeTestRule.setContent {
            HomeScreen()
        }

        // ローディング表示
        composeTestRule.onNodeWithTag("loading").assertIsDisplayed()

        // データ表示を待機
        composeTestRule.waitUntil(5000) {
            composeTestRule.onAllNodesWithTag("list_item").fetchSemanticsNodes().isNotEmpty()
        }

        composeTestRule.onAllNodesWithTag("list_item").assertCountEquals(10)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `onNodeWithText` | テキストで検索 |
| `onNodeWithTag` | TestTagで検索 |
| `performClick` | クリック操作 |
| `assertIsDisplayed` | 表示確認 |
| `waitUntil` | 非同期待機 |

- `ComposeTestRule`でComposableを単体テスト
- `testTag`でテスト用の識別子を付与
- `waitUntil`で非同期処理の完了を待機
- Hiltと組み合わせてDI込みのテスト

---

8種類のAndroidアプリテンプレート（テスト設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テストRobotパターン](https://zenn.dev/myougatheaxo/articles/android-compose-testing-robot-pattern-2026)
- [コルーチンテスト](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-testing-2026)
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-screenshot-comparison-2026)
