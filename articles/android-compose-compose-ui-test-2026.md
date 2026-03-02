---
title: "Compose UIテスト完全ガイド — ComposeTestRule/セマンティクス/スクリーンショット"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Compose UIテスト**（ComposeTestRule、セマンティクスマッチャー、操作シミュレーション、スクリーンショットテスト）を解説します。

---

## 基本テスト

```kotlin
@get:Rule
val composeTestRule = createComposeRule()

@Test
fun loginButton_displaysCorrectText() {
    composeTestRule.setContent {
        LoginScreen()
    }

    composeTestRule
        .onNodeWithText("ログイン")
        .assertIsDisplayed()
        .assertIsEnabled()

    composeTestRule
        .onNodeWithTag("email_input")
        .performTextInput("test@example.com")

    composeTestRule
        .onNodeWithTag("password_input")
        .performTextInput("password123")

    composeTestRule
        .onNodeWithText("ログイン")
        .performClick()
}
```

---

## セマンティクスマッチャー

```kotlin
@Test
fun itemList_showsCorrectCount() {
    val items = listOf("りんご", "バナナ", "みかん")

    composeTestRule.setContent {
        ItemList(items = items)
    }

    // 全アイテム表示確認
    items.forEach { item ->
        composeTestRule.onNodeWithText(item).assertIsDisplayed()
    }

    // ノード数確認
    composeTestRule
        .onAllNodesWithTag("list_item")
        .assertCountEquals(3)

    // スクロールして検証
    composeTestRule
        .onNodeWithTag("item_list")
        .performScrollToIndex(2)

    composeTestRule
        .onNodeWithText("みかん")
        .assertIsDisplayed()
}

@Test
fun toggleSwitch_changesState() {
    composeTestRule.setContent {
        SettingsScreen()
    }

    composeTestRule
        .onNodeWithText("通知")
        .onSiblings()
        .filterToOne(hasClickAction())
        .performClick()

    composeTestRule
        .onNode(hasTestTag("notification_toggle") and isToggleable())
        .assertIsOn()
}
```

---

## カスタムアサーション

```kotlin
fun SemanticsNodeInteraction.assertBackgroundColor(expected: Color): SemanticsNodeInteraction {
    val node = fetchSemanticsNode()
    val colors = node.config.getOrNull(SemanticsProperties.BackgroundColor)
    assert(colors == expected) { "Expected $expected but was $colors" }
    return this
}

@Test
fun errorState_showsRedBackground() {
    composeTestRule.setContent {
        ErrorBanner(message = "エラー発生")
    }

    composeTestRule
        .onNodeWithText("エラー発生")
        .assertIsDisplayed()
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| ルール | `createComposeRule()` |
| 検索 | `onNodeWithText`, `onNodeWithTag` |
| 操作 | `performClick`, `performTextInput` |
| 検証 | `assertIsDisplayed`, `assertIsEnabled` |

- `createComposeRule()`でCompose UIテスト環境構築
- セマンティクスベースのノード検索で安定テスト
- `performClick`/`performTextInput`で操作シミュレーション
- `testTag`でテスト専用の識別子を付与

---

8種類のAndroidアプリテンプレート（テスト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [ViewModel Testing](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-testing-2026)
- [Coroutine Testing](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-testing-2026)
