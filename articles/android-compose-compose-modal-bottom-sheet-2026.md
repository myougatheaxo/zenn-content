---
title: "Compose ModalBottomSheet完全ガイド — M3BottomSheet/状態管理/フルスクリーン"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose ModalBottomSheet**（Material3 ModalBottomSheet、SheetState、スワイプ制御、フルスクリーン）を解説します。

---

## 基本ModalBottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BottomSheetDemo() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()

    Scaffold { padding ->
        Button(
            onClick = { showSheet = true },
            modifier = Modifier.padding(padding).padding(16.dp)
        ) { Text("シートを開く") }
    }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("オプション", style = MaterialTheme.typography.titleLarge)
                Spacer(Modifier.height(16.dp))
                ListItem(
                    headlineContent = { Text("編集") },
                    leadingContent = { Icon(Icons.Default.Edit, null) },
                    modifier = Modifier.clickable { showSheet = false }
                )
                ListItem(
                    headlineContent = { Text("共有") },
                    leadingContent = { Icon(Icons.Default.Share, null) },
                    modifier = Modifier.clickable { showSheet = false }
                )
                ListItem(
                    headlineContent = { Text("削除") },
                    leadingContent = { Icon(Icons.Default.Delete, null) },
                    modifier = Modifier.clickable { showSheet = false }
                )
                Spacer(Modifier.height(32.dp))
            }
        }
    }
}
```

---

## スキップ設定

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PartialSheet() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = false)

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            LazyColumn {
                items(30) { index ->
                    ListItem(
                        headlineContent = { Text("アイテム ${index + 1}") },
                        leadingContent = { Text("${index + 1}", style = MaterialTheme.typography.titleMedium) }
                    )
                }
            }
        }
    }
}
```

---

## プログラム制御

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProgrammaticSheet() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()
    val scope = rememberCoroutineScope()

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("確認", style = MaterialTheme.typography.titleLarge)
                Text("この操作を実行しますか？")
                Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.End) {
                    TextButton(onClick = {
                        scope.launch { sheetState.hide() }.invokeOnCompletion {
                            if (!sheetState.isVisible) showSheet = false
                        }
                    }) { Text("キャンセル") }
                    Button(onClick = {
                        // 処理実行
                        scope.launch { sheetState.hide() }.invokeOnCompletion {
                            if (!sheetState.isVisible) showSheet = false
                        }
                    }) { Text("実行") }
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
| `ModalBottomSheet` | M3ボトムシート |
| `rememberModalBottomSheetState` | 状態管理 |
| `skipPartiallyExpanded` | 中間状態スキップ |
| `sheetState.hide()` | プログラム的に閉じる |

- `ModalBottomSheet`はM3標準のボトムシート
- `onDismissRequest`でスワイプ/外側タップの閉じ処理
- `skipPartiallyExpanded = false`で中間展開状態を有効化
- `scope.launch { sheetState.hide() }`でプログラム的に制御

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
- [Compose Dialog](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dialog-2026)
- [Compose DropdownMenu](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dropdown-menu-2026)
