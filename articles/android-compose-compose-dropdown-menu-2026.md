---
title: "DropdownMenu完全ガイド — メニュー表示/アイテム選択/ネストメニュー"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**DropdownMenu**（メニュー表示、アイテム選択、アイコン付きメニュー、オーバーフローメニュー）を解説します。

---

## 基本DropdownMenu

```kotlin
@Composable
fun BasicDropdownMenu() {
    var expanded by remember { mutableStateOf(false) }
    var selectedOption by remember { mutableStateOf("選択してください") }

    Box {
        OutlinedButton(onClick = { expanded = true }) {
            Text(selectedOption)
            Icon(Icons.Default.ArrowDropDown, null)
        }
        DropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
            listOf("オプション1", "オプション2", "オプション3").forEach { option ->
                DropdownMenuItem(
                    text = { Text(option) },
                    onClick = {
                        selectedOption = option
                        expanded = false
                    }
                )
            }
        }
    }
}
```

---

## オーバーフローメニュー

```kotlin
@Composable
fun OverflowMenuExample() {
    var expanded by remember { mutableStateOf(false) }

    TopAppBar(
        title = { Text("設定") },
        actions = {
            IconButton(onClick = { expanded = true }) {
                Icon(Icons.Default.MoreVert, "メニュー")
            }
            DropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
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
                        textColor = Color.Red,
                        leadingIconColor = Color.Red
                    )
                )
            }
        }
    )
}
```

---

## ソート選択メニュー

```kotlin
@Composable
fun SortMenu(currentSort: String, onSortChanged: (String) -> Unit) {
    var expanded by remember { mutableStateOf(false) }
    val sortOptions = listOf("名前順", "日付順", "サイズ順", "種類順")

    Box {
        TextButton(onClick = { expanded = true }) {
            Icon(Icons.Default.Sort, null, Modifier.size(18.dp))
            Spacer(Modifier.width(4.dp))
            Text(currentSort)
        }
        DropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
            sortOptions.forEach { option ->
                DropdownMenuItem(
                    text = { Text(option) },
                    onClick = {
                        onSortChanged(option)
                        expanded = false
                    },
                    trailingIcon = {
                        if (option == currentSort) Icon(Icons.Default.Check, null)
                    }
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
| `DropdownMenu` | メニューコンテナ |
| `DropdownMenuItem` | メニューアイテム |
| `leadingIcon` | 先頭アイコン |
| `trailingIcon` | 末尾アイコン |

- `expanded`で表示/非表示制御
- `onDismissRequest`で外側タップ時に閉じる
- `leadingIcon`/`trailingIcon`でアイコン付きメニュー
- TopAppBarのオーバーフローメニューに活用

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ContextMenu](https://zenn.dev/myougatheaxo/articles/android-compose-compose-context-menu-2026)
- [TopAppBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-top-app-bar-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
