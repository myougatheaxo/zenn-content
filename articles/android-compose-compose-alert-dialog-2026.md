---
title: "AlertDialog完全ガイド — AlertDialog/BasicAlertDialog/カスタムDialog"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**AlertDialog**（AlertDialog、BasicAlertDialog、カスタムDialog、入力ダイアログ）を解説します。

---

## 基本AlertDialog

```kotlin
@Composable
fun AlertDialogExample() {
    var showDialog by remember { mutableStateOf(false) }

    Button(onClick = { showDialog = true }) { Text("削除") }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            icon = { Icon(Icons.Default.Warning, null) },
            title = { Text("削除確認") },
            text = { Text("このアイテムを削除しますか？この操作は取り消せません。") },
            confirmButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("削除", color = MaterialTheme.colorScheme.error)
                }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("キャンセル")
                }
            }
        )
    }
}
```

---

## 入力付きDialog

```kotlin
@Composable
fun InputDialog(
    onDismiss: () -> Unit,
    onConfirm: (String) -> Unit
) {
    var text by remember { mutableStateOf("") }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("新規フォルダ") },
        text = {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text("フォルダ名") },
                singleLine = true,
                modifier = Modifier.fillMaxWidth()
            )
        },
        confirmButton = {
            TextButton(
                onClick = { onConfirm(text) },
                enabled = text.isNotBlank()
            ) { Text("作成") }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("キャンセル") }
        }
    )
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
                    title = { Text("編集") },
                    navigationIcon = {
                        IconButton(onClick = onDismiss) {
                            Icon(Icons.Default.Close, "閉じる")
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
                    value = "",
                    onValueChange = {},
                    label = { Text("タイトル") },
                    modifier = Modifier.fillMaxWidth()
                )
                Spacer(Modifier.height(8.dp))
                OutlinedTextField(
                    value = "",
                    onValueChange = {},
                    label = { Text("説明") },
                    modifier = Modifier.fillMaxWidth().height(200.dp)
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
| `AlertDialog` | Material3ダイアログ |
| `Dialog` | カスタムダイアログ |
| `BasicAlertDialog` | 最小限ダイアログ |
| `DialogProperties` | 動作設定 |

- `AlertDialog`で確認/警告ダイアログ
- `Dialog`+`usePlatformDefaultWidth = false`でフルスクリーン
- 入力付きダイアログで名前/値の入力
- `icon`パラメータでアイコン付きダイアログ

---

8種類のAndroidアプリテンプレート（ダイアログ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
- [Snackbar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-snackbar-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
