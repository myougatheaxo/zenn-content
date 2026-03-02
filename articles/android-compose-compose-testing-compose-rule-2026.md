---
title: "Compose ComposeTestRule完全ガイド — createComposeRule/テストタグ/アサーション"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Compose ComposeTestRule**（createComposeRule、テストタグ、Finder、Assertion、Action）を解説します。

---

## 基本テスト

```kotlin
class ButtonTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun buttonClick_updatesCounter() {
        composeTestRule.setContent {
            var count by remember { mutableIntStateOf(0) }
            Column {
                Text("Count: $count", Modifier.testTag("counter"))
                Button(onClick = { count++ }, Modifier.testTag("increment")) {
                    Text("増加")
                }
            }
        }

        composeTestRule.onNodeWithTag("counter").assertTextEquals("Count: 0")
        composeTestRule.onNodeWithTag("increment").performClick()
        composeTestRule.onNodeWithTag("counter").assertTextEquals("Count: 1")
    }
}
```

---

## Finder一覧

```kotlin
@Test
fun finders() {
    composeTestRule.setContent { SampleScreen() }

    // テキストで検索
    composeTestRule.onNodeWithText("ログイン")
    composeTestRule.onNodeWithText("ログイン", substring = true)
    composeTestRule.onNodeWithText("ログイン", ignoreCase = true)

    // TestTagで検索
    composeTestRule.onNodeWithTag("login_button")

    // ContentDescriptionで検索
    composeTestRule.onNodeWithContentDescription("検索")

    // 複数ノード
    composeTestRule.onAllNodesWithTag("list_item").assertCountEquals(5)
    composeTestRule.onAllNodesWithText("アイテム")[0].assertIsDisplayed()

    // セマンティクスマッチャー
    composeTestRule.onNode(hasText("ログイン") and hasClickAction())
    composeTestRule.onNode(hasScrollAction())
}
```

---

## アクション+アサーション

```kotlin
@Test
fun textInput_and_validation() {
    composeTestRule.setContent { LoginScreen() }

    // テキスト入力
    composeTestRule.onNodeWithTag("email")
        .performTextInput("test@example.com")

    // テキストクリア+入力
    composeTestRule.onNodeWithTag("email")
        .performTextClearance()
        .performTextInput("new@example.com")

    // スクロール
    composeTestRule.onNodeWithTag("list")
        .performScrollToIndex(10)

    // スワイプ
    composeTestRule.onNodeWithTag("item")
        .performTouchInput { swipeLeft() }

    // アサーション
    composeTestRule.onNodeWithTag("error")
        .assertIsDisplayed()
        .assertTextEquals("入力エラー")

    composeTestRule.onNodeWithTag("submit")
        .assertIsEnabled()
        .assertHasClickAction()

    // 非表示確認
    composeTestRule.onNodeWithTag("loading")
        .assertDoesNotExist()
}

// 非同期待ち
@Test
fun asyncOperation() {
    composeTestRule.setContent { AsyncScreen() }

    composeTestRule.onNodeWithTag("load_button").performClick()

    // Idle待ち
    composeTestRule.waitForIdle()

    // 条件待ち（タイムアウト付き）
    composeTestRule.waitUntil(timeoutMillis = 5000) {
        composeTestRule.onAllNodesWithTag("result").fetchSemanticsNodes().isNotEmpty()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `createComposeRule` | テストルール作成 |
| `onNodeWithTag` | TestTag検索 |
| `performClick` | クリック操作 |
| `assertIsDisplayed` | 表示確認 |

- `Modifier.testTag("tag")`でテスト用識別子を付与
- `onNodeWith*`でFinder、`perform*`でAction、`assert*`でAssertion
- `waitUntil`で非同期処理の完了を待機
- `onAllNodesWithTag`で複数ノードのテスト

---

8種類のAndroidアプリテンプレート（テスト設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [Compose TestingRobot](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-robot-2026)
- [Compose ScreenshotTesting](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-screenshot-2026)
