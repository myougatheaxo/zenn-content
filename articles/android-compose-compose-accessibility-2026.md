---
title: "Compose Accessibility完全ガイド — セマンティクス/TalkBack/contentDescription/テスト"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

**Compose Accessibility**（セマンティクス、TalkBack対応、contentDescription、アクセシビリティテスト）を解説します。

---

## 基本セマンティクス

```kotlin
@Composable
fun AccessibleCardDemo() {
    // ❌ TalkBackで情報が断片化
    Card(Modifier.clickable { }) {
        Text("商品名")
        Text("¥1,000")
        Icon(Icons.Default.ShoppingCart, "カート")
    }

    // ✅ セマンティクスを統合
    Card(
        Modifier.semantics(mergeDescendants = true) { }
            .clickable(onClickLabel = "カートに追加") { }
    ) {
        Text("商品名")
        Text("¥1,000")
        Icon(Icons.Default.ShoppingCart, contentDescription = null) // 親で統合
    }
}

// contentDescriptionの正しい使い方
@Composable
fun IconButtonAccessibility() {
    // ❌ 不十分
    IconButton(onClick = {}) { Icon(Icons.Default.Delete, "削除") }

    // ✅ 状態も含める
    IconButton(
        onClick = {},
        modifier = Modifier.semantics { stateDescription = "選択済み" }
    ) { Icon(Icons.Default.Delete, "選択したアイテムを削除") }
}
```

---

## カスタムアクション

```kotlin
@Composable
fun SwipeableItemAccessibility(item: String, onDelete: () -> Unit, onArchive: () -> Unit) {
    ListItem(
        headlineContent = { Text(item) },
        modifier = Modifier.semantics {
            customActions = listOf(
                CustomAccessibilityAction("削除") { onDelete(); true },
                CustomAccessibilityAction("アーカイブ") { onArchive(); true }
            )
        }
    )
}

// ライブリージョン（動的更新の通知）
@Composable
fun CounterAccessibility() {
    var count by remember { mutableIntStateOf(0) }

    Column(Modifier.padding(16.dp)) {
        Text(
            "カウント: $count",
            modifier = Modifier.semantics { liveRegion = LiveRegionMode.Polite }
        )
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) { Text("-") }
            Button(onClick = { count++ }) { Text("+") }
        }
    }
}
```

---

## アクセシビリティテスト

```kotlin
@Test
fun testAccessibility() {
    composeTestRule.setContent {
        AccessibleCardDemo()
    }

    // contentDescription確認
    composeTestRule.onNodeWithContentDescription("選択したアイテムを削除")
        .assertExists()

    // セマンティクス確認
    composeTestRule.onNodeWithText("商品名")
        .assert(hasClickAction())

    // TalkBackでの読み上げ順序確認
    composeTestRule.onRoot()
        .printToLog("ACCESSIBILITY")
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `semantics` | セマンティクス設定 |
| `mergeDescendants` | 子要素統合 |
| `contentDescription` | アイコン説明 |
| `liveRegion` | 動的更新通知 |

- `mergeDescendants = true`でTalkBack読み上げを統合
- 装飾アイコンの`contentDescription`は`null`に
- `customActions`でスワイプ操作のアクセシビリティ代替
- `liveRegion`で動的に変わるテキストをTalkBackに通知

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Semantics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-semantics-2026)
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [Compose Locale](https://zenn.dev/myougatheaxo/articles/android-compose-compose-locale-2026)
