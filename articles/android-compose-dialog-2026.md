---
title: "ダイアログ実装ガイド — AlertDialog/DatePicker/カスタムダイアログ"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "dialog"]
published: true
---

## この記事で学べること

Composeの**ダイアログ**（AlertDialog、確認ダイアログ、カスタムダイアログ）を実装する方法を解説します。

---

## 基本のAlertDialog

```kotlin
@Composable
fun BasicDialog() {
    var showDialog by remember { mutableStateOf(false) }

    Button(onClick = { showDialog = true }) {
        Text("ダイアログを開く")
    }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("確認") },
            text = { Text("この操作を実行しますか？") },
            confirmButton = {
                TextButton(onClick = {
                    // 確認処理
                    showDialog = false
                }) {
                    Text("OK")
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

## 削除確認ダイアログ

```kotlin
@Composable
fun DeleteConfirmDialog(
    itemName: String,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        icon = {
            Icon(
                Icons.Default.Warning,
                contentDescription = null,
                tint = MaterialTheme.colorScheme.error
            )
        },
        title = { Text("削除の確認") },
        text = { Text("「$itemName」を削除しますか？この操作は取り消せません。") },
        confirmButton = {
            Button(
                onClick = onConfirm,
                colors = ButtonDefaults.buttonColors(
                    containerColor = MaterialTheme.colorScheme.error
                )
            ) {
                Text("削除")
            }
        },
        dismissButton = {
            OutlinedButton(onClick = onDismiss) {
                Text("キャンセル")
            }
        }
    )
}
```

---

## 入力ダイアログ

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
            ) {
                Text("追加")
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

---

## カスタムダイアログ

```kotlin
@Composable
fun CustomDialog(onDismiss: () -> Unit) {
    Dialog(onDismissRequest = onDismiss) {
        Card(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            shape = RoundedCornerShape(16.dp)
        ) {
            Column(
                Modifier.padding(24.dp),
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Icon(
                    Icons.Default.CheckCircle,
                    contentDescription = null,
                    modifier = Modifier.size(64.dp),
                    tint = Color(0xFF4CAF50)
                )

                Spacer(Modifier.height(16.dp))
                Text("完了しました！", style = MaterialTheme.typography.headlineSmall)

                Spacer(Modifier.height(8.dp))
                Text(
                    "操作が正常に完了しました。",
                    style = MaterialTheme.typography.bodyMedium,
                    textAlign = TextAlign.Center
                )

                Spacer(Modifier.height(24.dp))
                Button(
                    onClick = onDismiss,
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text("閉じる")
                }
            }
        }
    }
}
```

---

## 選択ダイアログ

```kotlin
@Composable
fun SelectionDialog(
    options: List<String>,
    selectedIndex: Int,
    onSelect: (Int) -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("選択してください") },
        text = {
            LazyColumn {
                itemsIndexed(options) { index, option ->
                    Row(
                        Modifier
                            .fillMaxWidth()
                            .clickable { onSelect(index) }
                            .padding(vertical = 12.dp),
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        RadioButton(
                            selected = index == selectedIndex,
                            onClick = { onSelect(index) }
                        )
                        Spacer(Modifier.width(8.dp))
                        Text(option)
                    }
                }
            }
        },
        confirmButton = {
            TextButton(onClick = onDismiss) {
                Text("決定")
            }
        }
    )
}
```

---

## まとめ

- `AlertDialog`で基本的な確認/入力ダイアログ
- `Dialog`でフルカスタムレイアウト
- `onDismissRequest`で外部タップ時の処理
- 削除確認はエラーカラーで視覚的に警告
- 入力ダイアログは`enabled`でバリデーション
- 選択ダイアログは`RadioButton`で排他選択

---

8種類のAndroidアプリテンプレート（ダイアログUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomSheet実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-2026)
- [Snackbar実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-snackbar-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
