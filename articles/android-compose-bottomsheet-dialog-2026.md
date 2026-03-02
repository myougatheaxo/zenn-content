---
title: "BottomSheet/Dialog完全ガイド — ModalBottomSheet/AlertDialog/カスタム"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**BottomSheetとDialog**（ModalBottomSheet、AlertDialog、カスタムDialog、状態管理）を解説します。

---

## ModalBottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BottomSheetExample() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = false)

    Button(onClick = { showSheet = true }) { Text("BottomSheet表示") }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState,
            dragHandle = { BottomSheetDefaults.DragHandle() }
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("メニュー", style = MaterialTheme.typography.titleLarge)
                Spacer(Modifier.height(16.dp))

                listOf("編集", "共有", "削除").forEach { action ->
                    ListItem(
                        headlineContent = { Text(action) },
                        leadingContent = {
                            Icon(
                                when (action) {
                                    "編集" -> Icons.Default.Edit
                                    "共有" -> Icons.Default.Share
                                    else -> Icons.Default.Delete
                                }, null
                            )
                        },
                        modifier = Modifier.clickable { showSheet = false }
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
        icon = { Icon(Icons.Default.Warning, null, tint = MaterialTheme.colorScheme.error) },
        title = { Text("削除の確認") },
        text = { Text("このアイテムを削除しますか？この操作は元に戻せません。") },
        confirmButton = {
            Button(
                onClick = onConfirm,
                colors = ButtonDefaults.buttonColors(
                    containerColor = MaterialTheme.colorScheme.error
                )
            ) { Text("削除") }
        },
        dismissButton = {
            OutlinedButton(onClick = onDismiss) { Text("キャンセル") }
        }
    )
}
```

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

    Dialog(onDismissRequest = onDismiss) {
        Card(
            Modifier.fillMaxWidth().padding(16.dp),
            shape = RoundedCornerShape(16.dp)
        ) {
            Column(Modifier.padding(24.dp)) {
                Text(title, style = MaterialTheme.typography.titleLarge)
                Spacer(Modifier.height(16.dp))
                OutlinedTextField(
                    value = text,
                    onValueChange = { text = it },
                    label = { Text("入力") },
                    modifier = Modifier.fillMaxWidth(),
                    singleLine = true
                )
                Spacer(Modifier.height(24.dp))
                Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.End) {
                    TextButton(onClick = onDismiss) { Text("キャンセル") }
                    Spacer(Modifier.width(8.dp))
                    Button(onClick = { onConfirm(text) }, enabled = text.isNotBlank()) {
                        Text("OK")
                    }
                }
            }
        }
    }
}
```

---

## 状態管理パターン

```kotlin
// ViewModel でDialog状態を管理
@HiltViewModel
class ItemViewModel @Inject constructor() : ViewModel() {
    var dialogState by mutableStateOf<DialogState>(DialogState.Hidden)
        private set

    fun showDeleteDialog(itemId: String) {
        dialogState = DialogState.DeleteConfirm(itemId)
    }

    fun showEditDialog(item: Item) {
        dialogState = DialogState.Edit(item)
    }

    fun dismissDialog() { dialogState = DialogState.Hidden }
}

sealed interface DialogState {
    data object Hidden : DialogState
    data class DeleteConfirm(val itemId: String) : DialogState
    data class Edit(val item: Item) : DialogState
}
```

---

## まとめ

| コンポーネント | 用途 |
|---------------|------|
| `ModalBottomSheet` | メニュー/オプション |
| `AlertDialog` | 確認/警告 |
| `Dialog` | カスタムUI |
| sealed interface | 状態管理 |

- `ModalBottomSheet`でスワイプ可能なメニュー
- `AlertDialog`で確認ダイアログ
- カスタム`Dialog`で入力フォーム
- sealed interfaceでDialog状態を型安全に管理

---

8種類のAndroidアプリテンプレート（Dialog/BottomSheet実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [状態管理](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
