---
title: "ContextMenu完全ガイド — 長押しメニュー/DropdownMenu/カスタムメニュー"
emoji: "📌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**ContextMenu**（長押しメニュー、DropdownMenu、カスタムポジションメニュー）を解説します。

---

## 長押しContextMenu

```kotlin
@Composable
fun ContextMenuExample() {
    var expanded by remember { mutableStateOf(false) }

    Box {
        ListItem(
            headlineContent = { Text("アイテム") },
            modifier = Modifier.combinedClickable(
                onClick = {},
                onLongClick = { expanded = true }
            )
        )

        DropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            DropdownMenuItem(
                text = { Text("コピー") },
                onClick = { expanded = false },
                leadingIcon = { Icon(Icons.Default.ContentCopy, null) }
            )
            DropdownMenuItem(
                text = { Text("編集") },
                onClick = { expanded = false },
                leadingIcon = { Icon(Icons.Default.Edit, null) }
            )
            HorizontalDivider()
            DropdownMenuItem(
                text = { Text("削除") },
                onClick = { expanded = false },
                leadingIcon = { Icon(Icons.Default.Delete, null) },
                colors = MenuDefaults.itemColors(
                    textColor = MaterialTheme.colorScheme.error,
                    leadingIconColor = MaterialTheme.colorScheme.error
                )
            )
        }
    }
}
```

---

## リスト内ContextMenu

```kotlin
@Composable
fun ListWithContextMenu(items: List<Item>, onAction: (Item, String) -> Unit) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            var expanded by remember { mutableStateOf(false) }

            Box {
                ListItem(
                    headlineContent = { Text(item.title) },
                    supportingContent = { Text(item.description) },
                    trailingContent = {
                        IconButton(onClick = { expanded = true }) {
                            Icon(Icons.Default.MoreVert, "メニュー")
                        }
                    }
                )

                DropdownMenu(
                    expanded = expanded,
                    onDismissRequest = { expanded = false }
                ) {
                    listOf("共有", "お気に入り", "削除").forEach { action ->
                        DropdownMenuItem(
                            text = { Text(action) },
                            onClick = {
                                onAction(item, action)
                                expanded = false
                            }
                        )
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `DropdownMenu` | メニュー表示 |
| `DropdownMenuItem` | メニュー項目 |
| `combinedClickable` | 長押し検知 |
| `MoreVert` | メニューボタン |

- `combinedClickable`で長押しメニュー表示
- `DropdownMenu`でMaterial3メニュー
- `HorizontalDivider`でメニュー内区切り
- `MenuDefaults.itemColors`でカスタムカラー

---

8種類のAndroidアプリテンプレート（UI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Popup Menu](https://zenn.dev/myougatheaxo/articles/android-compose-popup-menu-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
- [SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-compose-swipe-dismiss-2026)
