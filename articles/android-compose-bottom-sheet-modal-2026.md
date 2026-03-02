---
title: "BottomSheet完全ガイド — Modal/Standard/PartiallyExpanded"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**BottomSheet**（ModalBottomSheet、SheetState、PartiallyExpanded）を解説します。

---

## ModalBottomSheet

```kotlin
@Composable
fun ModalBottomSheetExample() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()

    Button(onClick = { showSheet = true }) {
        Text("BottomSheetを開く")
    }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("メニュー", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(16.dp))

                ListItem(
                    headlineContent = { Text("共有") },
                    leadingContent = { Icon(Icons.Default.Share, null) },
                    modifier = Modifier.clickable { showSheet = false }
                )
                ListItem(
                    headlineContent = { Text("編集") },
                    leadingContent = { Icon(Icons.Default.Edit, null) },
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

## PartiallyExpanded（半開き状態）

```kotlin
@Composable
fun PartiallyExpandedSheet() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState(
        skipPartiallyExpanded = false // 半開き状態を有効化
    )

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            // 半開き時に見える部分
            Column(Modifier.padding(16.dp)) {
                Text("検索フィルター", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(8.dp))

                // フィルターチップ
                FlowRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                    listOf("新着順", "人気順", "価格順").forEach { filter ->
                        FilterChip(
                            selected = false,
                            onClick = { },
                            label = { Text(filter) }
                        )
                    }
                }

                // 展開時に見える部分
                Spacer(Modifier.height(16.dp))
                Text("詳細フィルター", style = MaterialTheme.typography.titleSmall)

                // 価格範囲
                var priceRange by remember { mutableStateOf(0f..10000f) }
                RangeSlider(
                    value = priceRange,
                    onValueChange = { priceRange = it },
                    valueRange = 0f..50000f
                )

                Spacer(Modifier.height(16.dp))
                Button(
                    onClick = { showSheet = false },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text("適用")
                }

                Spacer(Modifier.height(32.dp))
            }
        }
    }
}
```

---

## プログラムで開閉制御

```kotlin
@Composable
fun ProgrammaticSheet() {
    val scope = rememberCoroutineScope()
    val sheetState = rememberModalBottomSheetState()
    var showSheet by remember { mutableStateOf(false) }

    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        Button(onClick = { showSheet = true }) {
            Text("開く")
        }
        Button(onClick = {
            scope.launch { sheetState.hide() }.invokeOnCompletion {
                showSheet = false
            }
        }) {
            Text("閉じる")
        }
        Button(onClick = {
            scope.launch { sheetState.expand() }
        }) {
            Text("全開")
        }
    }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            // コンテンツ
        }
    }
}
```

---

## リスト付きBottomSheet

```kotlin
@Composable
fun ListBottomSheet(
    items: List<String>,
    onItemSelected: (String) -> Unit,
    onDismiss: () -> Unit
) {
    ModalBottomSheet(onDismissRequest = onDismiss) {
        Text(
            "選択してください",
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.padding(horizontal = 16.dp)
        )
        Spacer(Modifier.height(8.dp))

        LazyColumn {
            items(items) { item ->
                ListItem(
                    headlineContent = { Text(item) },
                    modifier = Modifier.clickable {
                        onItemSelected(item)
                        onDismiss()
                    }
                )
            }
        }

        Spacer(Modifier.height(32.dp))
    }
}
```

---

## まとめ

- `ModalBottomSheet`でモーダルBottomSheet
- `rememberModalBottomSheetState()`で状態管理
- `skipPartiallyExpanded = false`で半開き状態有効
- `sheetState.expand()`/`sheetState.hide()`でプログラム制御
- `onDismissRequest`で閉じる処理
- LazyColumnでスクロール可能なリスト表示

---

8種類のAndroidアプリテンプレート（BottomSheet設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomSheet基本ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-2026)
- [ダイアログ実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-dialog-2026)
- [ジェスチャーガイド](https://zenn.dev/myougatheaxo/articles/compose-gesture-touch-2026)
