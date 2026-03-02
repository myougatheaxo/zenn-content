---
title: "コンテキストメニュー実装 — 長押しメニュー/BottomActionSheet/ModalMenu"
emoji: "📑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "menu"]
published: true
---

## この記事で学べること

Composeでの**コンテキストメニュー**（長押しメニュー、BottomActionSheet、カスタムメニュー）を解説します。

---

## 長押しドロップダウンメニュー

```kotlin
@Composable
fun LongPressMenu(item: Item, onAction: (String) -> Unit) {
    var showMenu by remember { mutableStateOf(false) }

    Box {
        ListItem(
            headlineContent = { Text(item.title) },
            modifier = Modifier.combinedClickable(
                onClick = { onAction("open") },
                onLongClick = { showMenu = true }
            )
        )

        DropdownMenu(
            expanded = showMenu,
            onDismissRequest = { showMenu = false }
        ) {
            DropdownMenuItem(
                text = { Text("編集") },
                onClick = { showMenu = false; onAction("edit") },
                leadingIcon = { Icon(Icons.Default.Edit, null) }
            )
            DropdownMenuItem(
                text = { Text("共有") },
                onClick = { showMenu = false; onAction("share") },
                leadingIcon = { Icon(Icons.Default.Share, null) }
            )
            HorizontalDivider()
            DropdownMenuItem(
                text = { Text("削除", color = MaterialTheme.colorScheme.error) },
                onClick = { showMenu = false; onAction("delete") },
                leadingIcon = {
                    Icon(Icons.Default.Delete, null, tint = MaterialTheme.colorScheme.error)
                }
            )
        }
    }
}
```

---

## BottomSheet アクションメニュー

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ActionBottomSheet(
    item: Item,
    onDismiss: () -> Unit,
    onAction: (String) -> Unit
) {
    ModalBottomSheet(onDismissRequest = onDismiss) {
        Column(Modifier.padding(bottom = 32.dp)) {
            // ヘッダー
            ListItem(
                headlineContent = { Text(item.title, fontWeight = FontWeight.Bold) },
                supportingContent = { Text("操作を選択") }
            )

            HorizontalDivider()

            // アクション
            ListItem(
                headlineContent = { Text("編集") },
                leadingContent = { Icon(Icons.Default.Edit, null) },
                modifier = Modifier.clickable { onAction("edit"); onDismiss() }
            )
            ListItem(
                headlineContent = { Text("複製") },
                leadingContent = { Icon(Icons.Default.ContentCopy, null) },
                modifier = Modifier.clickable { onAction("copy"); onDismiss() }
            )
            ListItem(
                headlineContent = { Text("共有") },
                leadingContent = { Icon(Icons.Default.Share, null) },
                modifier = Modifier.clickable { onAction("share"); onDismiss() }
            )
            ListItem(
                headlineContent = {
                    Text("削除", color = MaterialTheme.colorScheme.error)
                },
                leadingContent = {
                    Icon(Icons.Default.Delete, null, tint = MaterialTheme.colorScheme.error)
                },
                modifier = Modifier.clickable { onAction("delete"); onDismiss() }
            )
        }
    }
}
```

---

## 選択モードバー

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SelectionModeTopBar(
    selectedCount: Int,
    onClose: () -> Unit,
    onDelete: () -> Unit,
    onShare: () -> Unit
) {
    TopAppBar(
        title = { Text("$selectedCount 件選択") },
        navigationIcon = {
            IconButton(onClick = onClose) {
                Icon(Icons.Default.Close, "選択解除")
            }
        },
        actions = {
            IconButton(onClick = onShare) {
                Icon(Icons.Default.Share, "共有")
            }
            IconButton(onClick = onDelete) {
                Icon(Icons.Default.Delete, "削除")
            }
        },
        colors = TopAppBarDefaults.topAppBarColors(
            containerColor = MaterialTheme.colorScheme.primaryContainer
        )
    )
}
```

---

## 使い方の使い分け

| パターン | 適用場面 |
|---------|---------|
| DropdownMenu | リスト項目の長押し、テキストの右クリック相当 |
| ModalBottomSheet | 多くのアクションがある場合、モバイルフレンドリー |
| TopBar選択モード | 複数選択時の一括操作 |
| AlertDialog | 確認が必要なアクション（削除等） |

---

## まとめ

- `combinedClickable(onLongClick)`で長押し検出
- `DropdownMenu`で軽量なコンテキストメニュー
- `ModalBottomSheet`でモバイル向けアクションシート
- 選択モードは`TopAppBar`のカラー変更で視覚的に区別
- 削除等の危険な操作は赤色で視覚的に警告

---

8種類のAndroidアプリテンプレート（メニューUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DropdownMenu/Tooltipガイド](https://zenn.dev/myougatheaxo/articles/android-compose-tooltip-menu-2026)
- [BottomSheet実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-2026)
- [ダイアログ実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-dialog-2026)
