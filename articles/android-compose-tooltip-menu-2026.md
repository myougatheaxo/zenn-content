---
title: "DropdownMenu・Tooltip実装ガイド — Composeでメニューとヒント表示"
emoji: "📌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**DropdownMenu**、**ExposedDropdownMenu**、**Tooltip**をComposeで実装する方法を解説します。

---

## DropdownMenu

```kotlin
@Composable
fun DropdownMenuExample() {
    var expanded by remember { mutableStateOf(false) }

    Box {
        IconButton(onClick = { expanded = true }) {
            Icon(Icons.Default.MoreVert, "メニュー")
        }

        DropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            DropdownMenuItem(
                text = { Text("編集") },
                onClick = { expanded = false },
                leadingIcon = { Icon(Icons.Default.Edit, null) }
            )
            DropdownMenuItem(
                text = { Text("共有") },
                onClick = { expanded = false },
                leadingIcon = { Icon(Icons.Default.Share, null) }
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

## ExposedDropdownMenu（選択式）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ExposedDropdownExample() {
    val options = listOf("日本語", "English", "中文", "한국어")
    var expanded by remember { mutableStateOf(false) }
    var selected by remember { mutableStateOf(options[0]) }

    ExposedDropdownMenuBox(
        expanded = expanded,
        onExpandedChange = { expanded = it }
    ) {
        OutlinedTextField(
            value = selected,
            onValueChange = {},
            readOnly = true,
            label = { Text("言語") },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
            modifier = Modifier
                .menuAnchor(MenuAnchorType.PrimaryNotEditable)
                .fillMaxWidth()
        )

        ExposedDropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            options.forEach { option ->
                DropdownMenuItem(
                    text = { Text(option) },
                    onClick = {
                        selected = option
                        expanded = false
                    }
                )
            }
        }
    }
}
```

---

## Tooltip

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TooltipExample() {
    TooltipBox(
        positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
        tooltip = {
            PlainTooltip {
                Text("お気に入りに追加")
            }
        },
        state = rememberTooltipState()
    ) {
        IconButton(onClick = { /* ... */ }) {
            Icon(Icons.Default.Favorite, "お気に入り")
        }
    }
}
```

---

## RichTooltip（リッチツールチップ）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RichTooltipExample() {
    TooltipBox(
        positionProvider = TooltipDefaults.rememberRichTooltipPositionProvider(),
        tooltip = {
            RichTooltip(
                title = { Text("カメラ設定") },
                action = {
                    TextButton(onClick = { /* ... */ }) {
                        Text("詳しく見る")
                    }
                }
            ) {
                Text("写真の解像度や保存先を設定できます。")
            }
        },
        state = rememberTooltipState()
    ) {
        IconButton(onClick = { /* ... */ }) {
            Icon(Icons.Default.CameraAlt, "カメラ")
        }
    }
}
```

---

## コンテキストメニュー

```kotlin
@Composable
fun ContextMenuExample(items: List<Item>) {
    LazyColumn {
        items(items) { item ->
            var showMenu by remember { mutableStateOf(false) }

            ListItem(
                headlineContent = { Text(item.title) },
                modifier = Modifier.combinedClickable(
                    onClick = { /* 通常タップ */ },
                    onLongClick = { showMenu = true }
                ),
                trailingContent = {
                    Box {
                        if (showMenu) {
                            DropdownMenu(
                                expanded = showMenu,
                                onDismissRequest = { showMenu = false }
                            ) {
                                DropdownMenuItem(
                                    text = { Text("コピー") },
                                    onClick = { showMenu = false }
                                )
                                DropdownMenuItem(
                                    text = { Text("削除") },
                                    onClick = { showMenu = false }
                                )
                            }
                        }
                    }
                }
            )
        }
    }
}
```

---

## まとめ

- `DropdownMenu`でポップアップメニュー
- `ExposedDropdownMenuBox`でセレクトボックス風UI
- `PlainTooltip` / `RichTooltip`でヒント表示
- `combinedClickable`の`onLongClick`でコンテキストメニュー
- `menuAnchor()`でメニューの位置を制御

---

8種類のAndroidアプリテンプレート（メニュー・ツールチップ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomSheet完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [ダイアログ完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-dialog-guide-2026)
