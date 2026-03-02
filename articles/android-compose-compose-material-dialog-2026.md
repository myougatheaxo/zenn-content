---
title: "Compose Material Dialog完全ガイド — AlertDialog/DatePickerDialog/TimePickerDialog/カスタムDialog"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Material Dialog**（AlertDialog、DatePickerDialog、TimePickerDialog、フルスクリーンDialog）を解説します。

---

## AlertDialog

```kotlin
@Composable
fun DeleteConfirmDialog(onConfirm: () -> Unit, onDismiss: () -> Unit) {
    AlertDialog(
        onDismissRequest = onDismiss,
        icon = { Icon(Icons.Default.Warning, contentDescription = null) },
        title = { Text("削除の確認") },
        text = { Text("このアイテムを完全に削除しますか？この操作は取り消せません。") },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text("削除", color = MaterialTheme.colorScheme.error)
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("キャンセル") }
        }
    )
}

@Composable
fun DialogDemo() {
    var showDialog by remember { mutableStateOf(false) }

    Button(onClick = { showDialog = true }) { Text("削除") }

    if (showDialog) {
        DeleteConfirmDialog(
            onConfirm = { showDialog = false },
            onDismiss = { showDialog = false }
        )
    }
}
```

---

## カスタムDialog

```kotlin
@Composable
fun InputDialog(onConfirm: (String) -> Unit, onDismiss: () -> Unit) {
    var text by remember { mutableStateOf("") }

    Dialog(onDismissRequest = onDismiss) {
        Card(
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            shape = RoundedCornerShape(16.dp)
        ) {
            Column(Modifier.padding(24.dp)) {
                Text("メモを追加", style = MaterialTheme.typography.headlineSmall)
                Spacer(Modifier.height(16.dp))
                OutlinedTextField(
                    value = text,
                    onValueChange = { text = it },
                    label = { Text("メモ") },
                    modifier = Modifier.fillMaxWidth()
                )
                Spacer(Modifier.height(24.dp))
                Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.End) {
                    TextButton(onClick = onDismiss) { Text("キャンセル") }
                    Spacer(Modifier.width(8.dp))
                    Button(onClick = { onConfirm(text) }) { Text("保存") }
                }
            }
        }
    }
}
```

---

## フルスクリーンDialog

```kotlin
@Composable
fun FullScreenDialog(onDismiss: () -> Unit) {
    Dialog(
        onDismissRequest = onDismiss,
        properties = DialogProperties(usePlatformDefaultWidth = false)
    ) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text("新規作成") },
                    navigationIcon = {
                        IconButton(onClick = onDismiss) {
                            Icon(Icons.Default.Close, contentDescription = "閉じる")
                        }
                    },
                    actions = {
                        TextButton(onClick = onDismiss) { Text("保存") }
                    }
                )
            }
        ) { padding ->
            Column(Modifier.padding(padding).padding(16.dp)) {
                OutlinedTextField(
                    value = "", onValueChange = {},
                    label = { Text("タイトル") },
                    modifier = Modifier.fillMaxWidth()
                )
                Spacer(Modifier.height(16.dp))
                OutlinedTextField(
                    value = "", onValueChange = {},
                    label = { Text("内容") },
                    modifier = Modifier.fillMaxWidth().weight(1f),
                    minLines = 10
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
| `AlertDialog` | 確認ダイアログ |
| `Dialog` | カスタムダイアログ |
| `DialogProperties` | ダイアログ設定 |
| `usePlatformDefaultWidth` | 幅制御 |

- `AlertDialog`でシンプルな確認/選択ダイアログ
- `Dialog`で自由なレイアウトのカスタムダイアログ
- `usePlatformDefaultWidth = false`でフルスクリーン
- `onDismissRequest`で外部タップ/戻るボタン処理

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
- [Compose Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-snackbar-2026)
- [Compose DatePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-date-picker-2026)
