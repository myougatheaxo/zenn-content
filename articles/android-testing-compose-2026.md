---
title: "ComposeのUIテスト入門 — テスト駆動で信頼性の高いUIを作る"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

Compose UIの**テスト方法**（テストルール、セマンティクス、アサーション）を解説します。

---

## 依存関係

```kotlin
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.7.6")
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.7.6")
}
```

---

## 基本のComposeテスト

```kotlin
@RunWith(AndroidJUnit4::class)
class CounterTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun counter_increments_on_button_click() {
        composeTestRule.setContent {
            var count by remember { mutableIntStateOf(0) }
            Column {
                Text("Count: $count", Modifier.testTag("counter"))
                Button(onClick = { count++ }) { Text("増やす") }
            }
        }

        composeTestRule.onNodeWithTag("counter").assertTextEquals("Count: 0")
        composeTestRule.onNodeWithText("増やす").performClick()
        composeTestRule.onNodeWithTag("counter").assertTextEquals("Count: 1")
    }
}
```

---

## ノードの検索

```kotlin
@Test
fun find_nodes() {
    composeTestRule.setContent { MyScreen() }

    composeTestRule.onNodeWithText("タイトル")
    composeTestRule.onNodeWithTag("submit_button")
    composeTestRule.onNodeWithContentDescription("削除ボタン")
    composeTestRule.onAllNodesWithText("アイテム").assertCountEquals(3)
    composeTestRule.onNode(hasText("送信") and hasClickAction())
}
```

---

## アサーション

```kotlin
@Test
fun assertions() {
    composeTestRule.setContent { MyScreen() }

    composeTestRule.onNodeWithText("タイトル").assertIsDisplayed()
    composeTestRule.onNodeWithTag("error").assertDoesNotExist()
    composeTestRule.onNodeWithText("送信").assertIsEnabled()
    composeTestRule.onNodeWithText("オプションA").assertIsSelected()
    composeTestRule.onNodeWithTag("input").assertTextContains("入力済み")
}
```

---

## アクション

```kotlin
@Test
fun user_actions() {
    composeTestRule.setContent { MyScreen() }

    composeTestRule.onNodeWithText("ボタン").performClick()
    composeTestRule.onNodeWithTag("email_input").performTextInput("test@example.com")
    composeTestRule.onNodeWithTag("email_input").performTextClearance()
    composeTestRule.onNodeWithTag("list").performScrollToIndex(10)
    composeTestRule.onNodeWithTag("card").performTouchInput { swipeLeft() }
}
```

---

## リストのテスト

```kotlin
@Test
fun list_displays_items() {
    val items = (1..20).map { Item(id = it, title = "Item $it") }

    composeTestRule.setContent {
        LazyColumn(Modifier.testTag("item_list")) {
            items(items) { item ->
                ListItem(
                    headlineContent = { Text(item.title) },
                    modifier = Modifier.testTag("item_${item.id}")
                )
            }
        }
    }

    composeTestRule.onNodeWithText("Item 1").assertIsDisplayed()
    composeTestRule.onNodeWithTag("item_list").performScrollToIndex(19)
    composeTestRule.onNodeWithText("Item 20").assertIsDisplayed()
}
```

---

## 非同期テスト

```kotlin
@Test
fun loading_then_content() {
    composeTestRule.setContent {
        var isLoading by remember { mutableStateOf(true) }
        LaunchedEffect(Unit) { delay(1000); isLoading = false }
        if (isLoading) CircularProgressIndicator(Modifier.testTag("loading"))
        else Text("完了", Modifier.testTag("content"))
    }

    composeTestRule.onNodeWithTag("loading").assertIsDisplayed()
    composeTestRule.mainClock.advanceTimeBy(1100)
    composeTestRule.onNodeWithTag("content").assertIsDisplayed()
}
```

---

## まとめ

- `createComposeRule()`でテストルール作成
- `onNodeWithText`/`onNodeWithTag`でノード検索
- `assertIsDisplayed()`/`assertDoesNotExist()`で表示確認
- `performClick()`/`performTextInput()`でユーザー操作
- `performScrollToIndex()`でリストスクロール
- `mainClock.advanceTimeBy()`で時間制御

---

8種類のAndroidアプリテンプレート（テスト追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
- [アクセシビリティガイド](https://zenn.dev/myougatheaxo/articles/compose-accessibility-2026)
