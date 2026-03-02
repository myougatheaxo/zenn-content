---
title: "ComposeTestRule完全ガイド — アサーション/アクション/待機"
emoji: "✅"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**ComposeTestRule**の詳細（ノード検索、アサーション、アクション、非同期待機）を解説します。

---

## ノード検索メソッド

```kotlin
@get:Rule
val composeTestRule = createComposeRule()

@Test
fun nodeFinderExamples() {
    composeTestRule.setContent { MyScreen() }

    // テキストで検索
    composeTestRule.onNodeWithText("送信")
    composeTestRule.onNodeWithText("送信", substring = true)
    composeTestRule.onNodeWithText("送信", ignoreCase = true)

    // testTagで検索
    composeTestRule.onNodeWithTag("email_field")

    // contentDescriptionで検索
    composeTestRule.onNodeWithContentDescription("戻る")

    // 複数ノード
    composeTestRule.onAllNodesWithText("アイテム")
    composeTestRule.onAllNodesWithTag("list_item")

    // 複合条件
    composeTestRule.onNode(
        hasText("送信") and hasClickAction()
    )
    composeTestRule.onNode(
        hasTestTag("item") and hasText("Kotlin")
    )
}
```

---

## アサーション

```kotlin
@Test
fun assertionExamples() {
    composeTestRule.setContent { MyScreen() }

    // 存在確認
    composeTestRule.onNodeWithText("タイトル").assertExists()
    composeTestRule.onNodeWithText("非表示").assertDoesNotExist()

    // 表示確認
    composeTestRule.onNodeWithText("タイトル").assertIsDisplayed()
    composeTestRule.onNodeWithText("隠れている").assertIsNotDisplayed()

    // 有効/無効
    composeTestRule.onNodeWithTag("submit").assertIsEnabled()
    composeTestRule.onNodeWithTag("submit").assertIsNotEnabled()

    // 選択状態
    composeTestRule.onNodeWithTag("checkbox").assertIsOn()
    composeTestRule.onNodeWithTag("checkbox").assertIsOff()
    composeTestRule.onNodeWithTag("option").assertIsSelected()

    // テキスト内容
    composeTestRule.onNodeWithTag("input").assertTextEquals("Hello")
    composeTestRule.onNodeWithTag("input").assertTextContains("Hel")

    // 子要素数
    composeTestRule.onNodeWithTag("list").onChildren().assertCountEquals(5)
}
```

---

## ユーザーアクション

```kotlin
@Test
fun actionExamples() {
    composeTestRule.setContent { MyScreen() }

    // クリック
    composeTestRule.onNodeWithText("送信").performClick()

    // テキスト入力
    composeTestRule.onNodeWithTag("email").performTextInput("user@test.com")
    composeTestRule.onNodeWithTag("email").performTextClearance()
    composeTestRule.onNodeWithTag("email").performTextReplacement("new@test.com")

    // スクロール
    composeTestRule.onNodeWithTag("list").performScrollToIndex(10)
    composeTestRule.onNodeWithTag("list").performScrollToNode(hasText("Target"))

    // スワイプ
    composeTestRule.onNodeWithTag("item").performTouchInput {
        swipeLeft()
        swipeRight()
        swipeUp()
        swipeDown()
    }

    // ロングクリック
    composeTestRule.onNodeWithTag("item").performTouchInput {
        longClick()
    }
}
```

---

## 非同期テスト

```kotlin
@Test
fun asyncTestExample() {
    composeTestRule.setContent {
        var isLoading by remember { mutableStateOf(true) }

        LaunchedEffect(Unit) {
            delay(1000)
            isLoading = false
        }

        if (isLoading) {
            CircularProgressIndicator(Modifier.testTag("loading"))
        } else {
            Text("完了", Modifier.testTag("result"))
        }
    }

    // ローディング表示確認
    composeTestRule.onNodeWithTag("loading").assertIsDisplayed()

    // メインクロックを進める
    composeTestRule.mainClock.advanceTimeBy(1100)

    // 結果表示確認
    composeTestRule.onNodeWithTag("result").assertIsDisplayed()

    // waitUntilで条件待ち
    composeTestRule.waitUntil(timeoutMillis = 5000) {
        composeTestRule.onAllNodesWithTag("result").fetchSemanticsNodes().isNotEmpty()
    }
}
```

---

## まとめ

- `onNodeWithText`/`onNodeWithTag`/`onNodeWithContentDescription`で検索
- `hasText() and hasClickAction()`で複合条件
- `assertIsDisplayed()`/`assertIsEnabled()`でUI状態検証
- `performClick()`/`performTextInput()`でユーザー操作
- `mainClock.advanceTimeBy()`でアニメーション/遅延を進める
- `waitUntil`で非同期処理の完了待ち

---

8種類のAndroidアプリテンプレート（テスト実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [UIテストガイド](https://zenn.dev/myougatheaxo/articles/android-compose-testing-ui-2026)
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-screenshot-2026)
- [ViewModel単体テスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-viewmodel-2026)
