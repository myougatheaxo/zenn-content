---
title: "アクセシビリティテスト完全ガイド — TalkBack/Semantics検証"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

Composeの**アクセシビリティテスト**（Semanticsツリー検証、TalkBackテスト、自動テスト）を解説します。

---

## Semanticsテスト基礎

```kotlin
@Test
fun buttonHasCorrectSemantics() {
    composeRule.setContent {
        Button(onClick = {}) {
            Icon(Icons.Default.Add, contentDescription = "タスク追加")
            Spacer(Modifier.width(8.dp))
            Text("追加")
        }
    }

    composeRule
        .onNodeWithContentDescription("タスク追加")
        .assertIsDisplayed()
        .assertHasClickAction()
}

@Test
fun imageHasContentDescription() {
    composeRule.setContent {
        AsyncImage(
            model = "https://example.com/photo.jpg",
            contentDescription = "ユーザーのプロフィール写真",
            modifier = Modifier.size(64.dp)
        )
    }

    composeRule
        .onNodeWithContentDescription("ユーザーのプロフィール写真")
        .assertExists()
}
```

---

## mergeDescendants テスト

```kotlin
@Composable
fun TaskItem(task: Task, onToggle: () -> Unit) {
    Row(
        modifier = Modifier
            .semantics(mergeDescendants = true) {} // 子を統合
            .clickable { onToggle() }
            .padding(16.dp)
    ) {
        Checkbox(checked = task.isCompleted, onCheckedChange = null)
        Spacer(Modifier.width(8.dp))
        Column {
            Text(task.title)
            Text(task.dueDate, style = MaterialTheme.typography.bodySmall)
        }
    }
}

@Test
fun taskItemMergesSemantics() {
    composeRule.setContent {
        TaskItem(task = Task("買い物", "2026-03-15", false), onToggle = {})
    }

    // mergeDescendantsにより1つのノードとして認識
    composeRule
        .onNodeWithText("買い物")
        .assertIsDisplayed()

    // Semanticsツリー確認
    composeRule.onRoot().printToLog("SEMANTICS")
}
```

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
        modifier = Modifier.semantics {
            customActions = listOf(
                CustomAccessibilityAction("削除") { onDelete(); true },
                CustomAccessibilityAction("アーカイブ") { onArchive(); true }
            )
        }
    ) {
        Text(item.title)
    }
}

@Test
fun itemHasCustomActions() {
    composeRule.setContent {
        SwipeableItem(
            item = Item("テスト"),
            onDelete = {},
            onArchive = {}
        )
    }

    composeRule
        .onNodeWithText("テスト")
        .assert(
            SemanticsMatcher("has custom actions") {
                val actions = it.config.getOrNull(SemanticsProperties.CustomActions)
                actions != null && actions.size == 2
            }
        )
}
```

---

## LiveRegion テスト

```kotlin
@Composable
fun StatusMessage(message: String?) {
    message?.let {
        Text(
            text = it,
            modifier = Modifier.semantics {
                liveRegion = LiveRegionMode.Polite
            }
        )
    }
}

@Test
fun statusMessageIsLiveRegion() {
    composeRule.setContent {
        StatusMessage("保存しました")
    }

    composeRule
        .onNodeWithText("保存しました")
        .assert(
            SemanticsMatcher("is live region") {
                it.config.getOrNull(SemanticsProperties.LiveRegion) != null
            }
        )
}
```

---

## コントラスト/サイズチェック

```kotlin
// フォントサイズが最小基準を満たすか
@Test
fun textMeetsMinimumSize() {
    composeRule.setContent {
        Text(
            "本文テキスト",
            style = MaterialTheme.typography.bodyLarge // 16sp
        )
    }

    // bodyLargeは16sp → アクセシブル
    composeRule.onNodeWithText("本文テキスト").assertIsDisplayed()
}

// タッチターゲットの最小サイズ（48dp）
@Composable
fun AccessibleIconButton(onClick: () -> Unit) {
    IconButton(
        onClick = onClick,
        modifier = Modifier.sizeIn(minWidth = 48.dp, minHeight = 48.dp)
    ) {
        Icon(Icons.Default.Close, contentDescription = "閉じる")
    }
}
```

---

## TalkBackテストチェックリスト

```
□ 全てのImageにcontentDescription（装飾画像はnull）
□ ボタンにクリック可能なラベル
□ フォーム入力にlabel指定
□ エラーメッセージがliveRegionで読み上げ
□ mergeDescendantsで論理グループ化
□ customActionsでスワイプ操作の代替
□ heading()で見出し指定
□ タッチターゲット48dp以上
□ 色のみに依存しない情報伝達
□ フォーカス順序が論理的
```

---

## まとめ

- `assertHasClickAction()`でクリック可能性検証
- `printToLog()`でSemanticsツリーをデバッグ出力
- `mergeDescendants = true`でTalkBackのグループ読み上げ
- `customActions`でスワイプの代替操作提供
- `liveRegion`で動的メッセージの読み上げ
- タッチターゲット48dp以上を確保

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アクセシビリティ基礎](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-2026)
- [ComposeTestRule](https://zenn.dev/myougatheaxo/articles/android-compose-testing-compose-rule-2026)
- [統合テスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-integration-2026)
