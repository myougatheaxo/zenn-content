---
title: "Compose UIテスト入門 — ComposeTestRuleで画面をテストする"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

Compose UIテストの書き方を解説します。**ComposeTestRule**を使えば、ボタンクリックやテキスト確認を自動テストできます。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}
```

---

## 基本のテスト

```kotlin
@RunWith(AndroidJUnit4::class)
class CounterScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun counter_increments_on_click() {
        composeTestRule.setContent {
            MyAppTheme {
                CounterScreen()
            }
        }

        // 初期値の確認
        composeTestRule.onNodeWithText("Count: 0").assertIsDisplayed()

        // ボタンをクリック
        composeTestRule.onNodeWithText("Count: 0").performClick()

        // 値が増えたか確認
        composeTestRule.onNodeWithText("Count: 1").assertIsDisplayed()
    }
}
```

---

## ノードの検索方法

```kotlin
// テキストで検索
composeTestRule.onNodeWithText("保存")

// contentDescriptionで検索
composeTestRule.onNodeWithContentDescription("削除ボタン")

// testTagで検索（推奨）
composeTestRule.onNodeWithTag("save_button")
```

### testTagの設定

```kotlin
Button(
    onClick = { },
    modifier = Modifier.testTag("save_button")
) {
    Text("保存")
}
```

`testTag`は**UIが変わってもテストが壊れにくい**のでおすすめ。

---

## テキスト入力テスト

```kotlin
@Test
fun search_filters_results() {
    composeTestRule.setContent {
        SearchScreen()
    }

    // テキスト入力
    composeTestRule
        .onNodeWithTag("search_field")
        .performTextInput("Kotlin")

    // フィルタ結果の確認
    composeTestRule
        .onNodeWithText("Kotlin入門")
        .assertIsDisplayed()

    composeTestRule
        .onNodeWithText("Java入門")
        .assertDoesNotExist()
}
```

---

## スクロールテスト

```kotlin
@Test
fun lazy_column_shows_items() {
    composeTestRule.setContent {
        ItemListScreen(items = (1..100).map { "Item $it" })
    }

    // スクロールしてアイテムを表示
    composeTestRule
        .onNodeWithTag("item_list")
        .performScrollToIndex(50)

    composeTestRule
        .onNodeWithText("Item 51")
        .assertIsDisplayed()
}
```

---

## ダイアログのテスト

```kotlin
@Test
fun delete_dialog_shows_on_long_press() {
    composeTestRule.setContent {
        HabitItem(habit = Habit("読書"), onDelete = {})
    }

    // 削除ボタンをクリック
    composeTestRule
        .onNodeWithContentDescription("削除")
        .performClick()

    // ダイアログが表示される
    composeTestRule
        .onNodeWithText("削除の確認")
        .assertIsDisplayed()

    // キャンセルボタン
    composeTestRule
        .onNodeWithText("キャンセル")
        .performClick()

    // ダイアログが閉じる
    composeTestRule
        .onNodeWithText("削除の確認")
        .assertDoesNotExist()
}
```

---

## アサーション一覧

| メソッド | 用途 |
|---------|------|
| `assertIsDisplayed()` | 表示されている |
| `assertDoesNotExist()` | 存在しない |
| `assertIsEnabled()` | 有効状態 |
| `assertIsNotEnabled()` | 無効状態 |
| `assertIsOn()` | チェック/スイッチがON |
| `assertTextEquals("text")` | テキスト一致 |

---

## まとめ

- `createComposeRule()`でテスト環境を作成
- `onNodeWithText`/`onNodeWithTag`でUI要素を検索
- `performClick()`/`performTextInput()`で操作
- `assertIsDisplayed()`/`assertDoesNotExist()`で確認
- `testTag`で安定したテストを書く

---

8種類のAndroidアプリテンプレート（テスト追加可能なクリーン設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テスト入門（AI時代のAndroidテスト）](https://zenn.dev/myougatheaxo/articles/android-testing-ai-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [ダイアログ完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-dialog-bottomsheet-2026)
