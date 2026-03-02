---
title: "アクセシビリティ応用ガイド — カスタムアクション/トラバーサル順序/テスト"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

**アクセシビリティ応用**（カスタムアクション、トラバーサル順序、ライブリージョン、テスト、TalkBack対応）を解説します。

---

## カスタムアクション

```kotlin
@Composable
fun SwipeableItem(
    item: Item,
    onDelete: () -> Unit,
    onArchive: () -> Unit
) {
    Box(
        Modifier
            .fillMaxWidth()
            .semantics {
                customActions = listOf(
                    CustomAccessibilityAction("削除") { onDelete(); true },
                    CustomAccessibilityAction("アーカイブ") { onArchive(); true }
                )
            }
    ) {
        ListItem(
            headlineContent = { Text(item.title) },
            supportingContent = { Text(item.description) }
        )
    }
}
```

---

## トラバーサル順序

```kotlin
@Composable
fun CustomTraversalOrder() {
    val (first, second, third) = remember { FocusRequester.createRefs() }

    Column {
        // 通常と異なる順序でフォーカスを制御
        Text(
            "3番目に読まれる",
            Modifier.semantics { traversalIndex = 3f }
        )
        Text(
            "1番目に読まれる",
            Modifier.semantics { traversalIndex = 1f }
        )
        Text(
            "2番目に読まれる",
            Modifier.semantics { traversalIndex = 2f }
        )
    }
}
```

---

## ライブリージョン（動的更新通知）

```kotlin
@Composable
fun TimerDisplay(remainingSeconds: Int) {
    Text(
        text = "${remainingSeconds}秒",
        modifier = Modifier.semantics {
            liveRegion = LiveRegionMode.Polite  // 変更時にTalkBackで読み上げ
            contentDescription = "残り${remainingSeconds}秒"
        },
        style = MaterialTheme.typography.displayLarge
    )
}

@Composable
fun ErrorBanner(message: String?) {
    message?.let {
        Surface(
            color = MaterialTheme.colorScheme.errorContainer,
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    liveRegion = LiveRegionMode.Assertive  // 即座に読み上げ
                }
        ) {
            Text(it, Modifier.padding(16.dp), color = MaterialTheme.colorScheme.onErrorContainer)
        }
    }
}
```

---

## semanticsのmerge/clear

```kotlin
@Composable
fun ClickableCard(title: String, subtitle: String, onClick: () -> Unit) {
    Card(
        onClick = onClick,
        modifier = Modifier.semantics(mergeDescendants = true) {
            // カード全体を1つの要素としてTalkBackに認識させる
            contentDescription = "$title。$subtitle"
        }
    ) {
        Column(Modifier.padding(16.dp)) {
            Text(title, style = MaterialTheme.typography.titleMedium)
            Text(subtitle, style = MaterialTheme.typography.bodySmall)
        }
    }
}

// 装飾的要素を非表示
Icon(
    Icons.Default.ChevronRight,
    contentDescription = null,  // TalkBackで無視
    modifier = Modifier.clearAndSetSemantics { }
)
```

---

## アクセシビリティテスト

```kotlin
@Test
fun cardHasCorrectSemantics() {
    composeTestRule.setContent {
        ClickableCard("タイトル", "サブタイトル", onClick = {})
    }

    composeTestRule
        .onNodeWithContentDescription("タイトル。サブタイトル")
        .assertIsDisplayed()
        .assertHasClickAction()
}

@Test
fun timerUpdatesLiveRegion() {
    composeTestRule.setContent {
        TimerDisplay(remainingSeconds = 30)
    }

    composeTestRule
        .onNodeWithContentDescription("残り30秒")
        .assertIsDisplayed()
}

@Test
fun customActionsAvailable() {
    composeTestRule.setContent {
        SwipeableItem(testItem, onDelete = {}, onArchive = {})
    }

    composeTestRule
        .onNodeWithText(testItem.title)
        .assert(
            SemanticsMatcher("has delete action") {
                it.config.getOrNull(SemanticsActions.CustomActions)
                    ?.any { action -> action.label == "削除" } == true
            }
        )
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| カスタムアクション | `CustomAccessibilityAction` |
| トラバーサル順序 | `traversalIndex` |
| 動的更新通知 | `liveRegion` |
| 要素マージ | `mergeDescendants` |
| 要素非表示 | `clearAndSetSemantics` |

- `customActions`でスワイプ操作のアクセシブル代替
- `traversalIndex`で読み上げ順序を制御
- `liveRegion`で動的コンテンツ変更を通知
- `mergeDescendants`で論理的なグループ化

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アクセシビリティ基本](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-basics-2026)
- [アクセシビリティテスト](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-testing-2026)
- [Compose UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-ui-2026)
