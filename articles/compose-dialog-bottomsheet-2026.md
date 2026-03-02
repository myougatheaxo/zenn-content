---
title: "Composeダイアログ完全ガイド — AlertDialog, BottomSheet, Snackbarの使い分け"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

ユーザーに確認を求める、選択肢を提示する、結果を通知する。これらの**ダイアログ系UI**をComposeで実装する方法を解説します。

---

## AlertDialog（確認ダイアログ）

```kotlin
@Composable
fun DeleteConfirmDialog(
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("削除の確認") },
        text = { Text("この項目を削除しますか？この操作は取り消せません。") },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text("削除", color = MaterialTheme.colorScheme.error)
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("キャンセル")
            }
        }
    )
}
```

### 使い方

```kotlin
@Composable
fun HabitItem(habit: Habit, onDelete: () -> Unit) {
    var showDialog by remember { mutableStateOf(false) }

    Card {
        Row {
            Text(habit.name)
            IconButton(onClick = { showDialog = true }) {
                Icon(Icons.Default.Delete, "削除")
            }
        }
    }

    if (showDialog) {
        DeleteConfirmDialog(
            onConfirm = {
                onDelete()
                showDialog = false
            },
            onDismiss = { showDialog = false }
        )
    }
}
```

---

## ModalBottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SortBottomSheet(
    onDismiss: () -> Unit,
    onSortSelected: (SortType) -> Unit
) {
    val sheetState = rememberModalBottomSheetState()

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState
    ) {
        Column(Modifier.padding(16.dp)) {
            Text(
                "並び替え",
                style = MaterialTheme.typography.titleMedium,
                modifier = Modifier.padding(bottom = 16.dp)
            )

            SortType.entries.forEach { sortType ->
                ListItem(
                    headlineContent = { Text(sortType.label) },
                    leadingContent = { Icon(sortType.icon, null) },
                    modifier = Modifier.clickable {
                        onSortSelected(sortType)
                        onDismiss()
                    }
                )
            }

            Spacer(Modifier.height(32.dp))
        }
    }
}
```

BottomSheetは選択肢が3つ以上あるとき、AlertDialogより使いやすい。

---

## Snackbar（一時的な通知）

```kotlin
@Composable
fun MainScreen(viewModel: HabitViewModel) {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(habits) { habit ->
                HabitItem(
                    habit = habit,
                    onDelete = {
                        viewModel.deleteHabit(habit)
                        scope.launch {
                            val result = snackbarHostState.showSnackbar(
                                message = "${habit.name}を削除しました",
                                actionLabel = "元に戻す",
                                duration = SnackbarDuration.Short
                            )
                            if (result == SnackbarResult.ActionPerformed) {
                                viewModel.undoDelete(habit)
                            }
                        }
                    }
                )
            }
        }
    }
}
```

**元に戻す（Undo）**パターンは、Snackbar + `actionLabel`で実装。

---

## 使い分けガイド

| シーン | UI |
|--------|-----|
| 確認が必要（削除、送信） | AlertDialog |
| 選択肢が3つ以上 | BottomSheet |
| 結果の通知 | Snackbar |
| 入力が必要 | AlertDialog + TextField |
| 設定変更 | BottomSheet |

---

## カスタムDialog

```kotlin
@Composable
fun InputDialog(
    title: String,
    onConfirm: (String) -> Unit,
    onDismiss: () -> Unit
) {
    var text by remember { mutableStateOf("") }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(title) },
        text = {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text("入力") },
                singleLine = true,
                modifier = Modifier.fillMaxWidth()
            )
        },
        confirmButton = {
            TextButton(
                onClick = { onConfirm(text) },
                enabled = text.isNotBlank()
            ) { Text("追加") }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("キャンセル") }
        }
    )
}
```

---

## まとめ

- **AlertDialog** — 確認・入力（最も汎用的）
- **ModalBottomSheet** — 複数選択肢・設定変更
- **Snackbar** — 結果通知・Undoアクション
- 表示状態は`remember { mutableStateOf(false) }`で管理

---

8種類のAndroidアプリテンプレート（ダイアログ・Snackbar実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Compose入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
