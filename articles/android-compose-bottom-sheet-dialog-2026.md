---
title: "BottomSheet/Dialog完全ガイド — Modal/Standard/AlertDialog"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**BottomSheet/Dialog**（ModalBottomSheet、AlertDialog、DatePicker、TimePicker、カスタムDialog）を解説します。

---

## ModalBottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BottomSheetExample() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()

    Button(onClick = { showSheet = true }) {
        Text("メニューを開く")
    }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("メニュー", style = MaterialTheme.typography.headlineSmall)
                Spacer(Modifier.height(16.dp))

                listOf("編集", "コピー", "共有", "削除").forEach { item ->
                    ListItem(
                        headlineContent = { Text(item) },
                        leadingContent = {
                            Icon(
                                when (item) {
                                    "編集" -> Icons.Default.Edit
                                    "コピー" -> Icons.Default.ContentCopy
                                    "共有" -> Icons.Default.Share
                                    else -> Icons.Default.Delete
                                },
                                contentDescription = null
                            )
                        },
                        modifier = Modifier.clickable {
                            // アクション処理
                            showSheet = false
                        }
                    )
                }

                Spacer(Modifier.height(32.dp))
            }
        }
    }
}
```

---

## AlertDialog

```kotlin
@Composable
fun DeleteConfirmDialog(
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        icon = { Icon(Icons.Default.Delete, null) },
        title = { Text("削除確認") },
        text = { Text("このアイテムを削除しますか？この操作は取り消せません。") },
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

// 使用
@Composable
fun ItemScreen() {
    var showDialog by remember { mutableStateOf(false) }

    Button(onClick = { showDialog = true }) { Text("削除") }

    if (showDialog) {
        DeleteConfirmDialog(
            onConfirm = { /* 削除処理 */ showDialog = false },
            onDismiss = { showDialog = false }
        )
    }
}
```

---

## DatePickerDialog

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerExample() {
    var showPicker by remember { mutableStateOf(false) }
    var selectedDate by remember { mutableStateOf<Long?>(null) }

    OutlinedTextField(
        value = selectedDate?.let {
            Instant.ofEpochMilli(it).atZone(ZoneId.systemDefault())
                .format(DateTimeFormatter.ofPattern("yyyy/MM/dd"))
        } ?: "",
        onValueChange = {},
        label = { Text("日付") },
        readOnly = true,
        trailingIcon = {
            IconButton(onClick = { showPicker = true }) {
                Icon(Icons.Default.CalendarMonth, "日付選択")
            }
        },
        modifier = Modifier.fillMaxWidth()
    )

    if (showPicker) {
        val datePickerState = rememberDatePickerState(
            initialSelectedDateMillis = selectedDate
        )
        DatePickerDialog(
            onDismissRequest = { showPicker = false },
            confirmButton = {
                TextButton(onClick = {
                    selectedDate = datePickerState.selectedDateMillis
                    showPicker = false
                }) { Text("OK") }
            },
            dismissButton = {
                TextButton(onClick = { showPicker = false }) { Text("キャンセル") }
            }
        ) {
            DatePicker(state = datePickerState)
        }
    }
}
```

---

## カスタムDialog

```kotlin
@Composable
fun InputDialog(
    title: String,
    initialValue: String = "",
    onConfirm: (String) -> Unit,
    onDismiss: () -> Unit
) {
    var text by remember { mutableStateOf(initialValue) }

    Dialog(onDismissRequest = onDismiss) {
        Card(
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            shape = RoundedCornerShape(16.dp)
        ) {
            Column(Modifier.padding(24.dp)) {
                Text(title, style = MaterialTheme.typography.headlineSmall)
                Spacer(Modifier.height(16.dp))

                OutlinedTextField(
                    value = text,
                    onValueChange = { text = it },
                    label = { Text("入力") },
                    modifier = Modifier.fillMaxWidth()
                )

                Spacer(Modifier.height(24.dp))

                Row(
                    Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.End
                ) {
                    TextButton(onClick = onDismiss) { Text("キャンセル") }
                    Spacer(Modifier.width(8.dp))
                    Button(onClick = { onConfirm(text) }) { Text("OK") }
                }
            }
        }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `ModalBottomSheet` | メニュー/オプション |
| `AlertDialog` | 確認/警告 |
| `DatePickerDialog` | 日付選択 |
| `Dialog` | カスタムレイアウト |

- `rememberModalBottomSheetState`でスワイプ制御
- `AlertDialog`で確認/キャンセルパターン
- `DatePicker`/`TimePicker`で日時選択
- `Dialog`+`Card`でカスタムUI

---

8種類のAndroidアプリテンプレート（UI完備）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [TextField高度](https://zenn.dev/myougatheaxo/articles/android-compose-text-field-advanced-2026)
