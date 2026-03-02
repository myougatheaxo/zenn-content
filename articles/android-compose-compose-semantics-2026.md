---
title: "Semantics完全ガイド — contentDescription/Role/カスタムアクション/テスト"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

**Semantics**（contentDescription、Role、カスタムアクション、TalkBack対応、テスト用Semantics）を解説します。

---

## 基本Semantics

```kotlin
@Composable
fun SemanticsExample() {
    Column(Modifier.padding(16.dp)) {
        // contentDescription
        IconButton(
            onClick = {},
            modifier = Modifier.semantics { contentDescription = "お気に入りに追加" }
        ) {
            Icon(Icons.Default.Favorite, contentDescription = null)
        }

        // stateDescription
        var isOn by remember { mutableStateOf(false) }
        Switch(
            checked = isOn,
            onCheckedChange = { isOn = it },
            modifier = Modifier.semantics {
                stateDescription = if (isOn) "有効" else "無効"
            }
        )

        // clearAndSetSemantics
        Row(
            Modifier.clearAndSetSemantics {
                contentDescription = "評価: 4.5点 (5点中)"
            }
        ) {
            repeat(5) { index ->
                Icon(
                    if (index < 4) Icons.Default.Star else Icons.Default.StarBorder,
                    contentDescription = null,
                    tint = Color(0xFFFFD700)
                )
            }
        }
    }
}
```

---

## カスタムアクション

```kotlin
@Composable
fun CustomActionItem(item: Item, onDelete: () -> Unit, onArchive: () -> Unit) {
    ListItem(
        headlineContent = { Text(item.title) },
        modifier = Modifier.semantics {
            customActions = listOf(
                CustomAccessibilityAction("削除") { onDelete(); true },
                CustomAccessibilityAction("アーカイブ") { onArchive(); true }
            )
        }
    )
}

// heading/Role
@Composable
fun SectionHeader(title: String) {
    Text(
        text = title,
        style = MaterialTheme.typography.headlineSmall,
        modifier = Modifier
            .padding(16.dp)
            .semantics { heading() }
    )
}

@Composable
fun ClickableCard(onClick: () -> Unit) {
    Card(
        modifier = Modifier
            .clickable(onClick = onClick)
            .semantics { role = Role.Button }
    ) {
        Text("タップしてください", Modifier.padding(16.dp))
    }
}
```

---

## テスト用Semantics

```kotlin
@Composable
fun TestableScreen() {
    Column {
        Text(
            "タイトル",
            modifier = Modifier.testTag("screen_title")
        )
        LazyColumn(Modifier.testTag("item_list")) {
            items(10) { index ->
                ListItem(
                    headlineContent = { Text("Item $index") },
                    modifier = Modifier.testTag("item_$index")
                )
            }
        }
    }
}

// テスト
class ScreenTest {
    @get:Rule val rule = createComposeRule()

    @Test
    fun titleIsDisplayed() {
        rule.setContent { TestableScreen() }
        rule.onNodeWithTag("screen_title").assertIsDisplayed()
        rule.onNodeWithTag("item_list").assertExists()
        rule.onNodeWithTag("item_0").assertIsDisplayed()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `contentDescription` | スクリーンリーダー用説明 |
| `clearAndSetSemantics` | 子要素のSemantics統合 |
| `customActions` | カスタムアクション追加 |
| `testTag` | UIテスト用識別子 |

- `contentDescription`でTalkBack読み上げテキスト設定
- `clearAndSetSemantics`で複合要素をひとまとめに
- `customActions`でスワイプメニューにアクション追加
- `testTag`でUIテストから要素を特定

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アクセシビリティ](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-2026)
- [UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ui-test-2026)
- [フォーカス管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-2026)
