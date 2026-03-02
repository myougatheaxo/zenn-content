---
title: "StickyHeader完全ガイド — stickyHeader/グループリスト/連絡先UI"
emoji: "📌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**StickyHeader**（stickyHeader、グループ化リスト、連絡先風UI）を解説します。

---

## 基本stickyHeader

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun StickyHeaderExample() {
    val grouped = mapOf(
        "A" to listOf("Alice", "Andrew", "Anna"),
        "B" to listOf("Bob", "Brian"),
        "C" to listOf("Charlie", "Chris", "Claire"),
        "D" to listOf("David", "Diana")
    )

    LazyColumn(Modifier.fillMaxSize()) {
        grouped.forEach { (initial, names) ->
            stickyHeader {
                Surface(
                    color = MaterialTheme.colorScheme.surfaceVariant,
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text(
                        initial,
                        modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                        style = MaterialTheme.typography.titleSmall,
                        color = MaterialTheme.colorScheme.primary
                    )
                }
            }

            items(names) { name ->
                ListItem(
                    headlineContent = { Text(name) },
                    leadingContent = {
                        Box(
                            Modifier.size(40.dp).background(MaterialTheme.colorScheme.primaryContainer, CircleShape),
                            contentAlignment = Alignment.Center
                        ) { Text(name.first().toString()) }
                    }
                )
            }
        }
    }
}
```

---

## 日付グループ

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun DateGroupedList(messages: List<Message>) {
    val grouped = messages.groupBy { it.date }

    LazyColumn {
        grouped.forEach { (date, msgs) ->
            stickyHeader {
                Box(
                    Modifier.fillMaxWidth().background(MaterialTheme.colorScheme.background),
                    contentAlignment = Alignment.Center
                ) {
                    Surface(
                        shape = RoundedCornerShape(12.dp),
                        tonalElevation = 2.dp,
                        modifier = Modifier.padding(8.dp)
                    ) {
                        Text(date, Modifier.padding(horizontal = 12.dp, vertical = 4.dp), fontSize = 12.sp)
                    }
                }
            }
            items(msgs) { msg ->
                ListItem(
                    headlineContent = { Text(msg.text) },
                    supportingContent = { Text(msg.time) }
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `stickyHeader` | 固定ヘッダー |
| `groupBy` | データグループ化 |
| `Surface` | ヘッダー背景 |
| `ExperimentalFoundationApi` | 実験的API |

- `stickyHeader`でスクロール時に固定されるヘッダー
- `groupBy`でリストデータをカテゴリ分け
- 連絡先/チャット/設定画面で活用
- ヘッダーに`Surface`で背景色を付与

---

8種類のAndroidアプリテンプレート（リスト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
- [ExpandableList](https://zenn.dev/myougatheaxo/articles/android-compose-expandable-list-2026)
- [NestedScroll](https://zenn.dev/myougatheaxo/articles/android-compose-compose-nested-scroll-2026)
