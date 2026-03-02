---
title: "BottomSheet完全ガイド — ModalBottomSheet/SheetState/半展開/フルスクリーン"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**BottomSheet**（ModalBottomSheet、SheetState、半展開、フルスクリーンシート）を解説します。

---

## ModalBottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BottomSheetExample() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = false)

    Scaffold { padding ->
        Button(
            onClick = { showSheet = true },
            modifier = Modifier.padding(padding).padding(16.dp)
        ) { Text("シートを開く") }
    }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState,
            shape = RoundedCornerShape(topStart = 16.dp, topEnd = 16.dp)
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("設定", style = MaterialTheme.typography.headlineSmall)
                Spacer(Modifier.height(16.dp))

                listOf("通知", "テーマ", "言語", "バージョン情報").forEach { item ->
                    ListItem(
                        headlineContent = { Text(item) },
                        leadingContent = { Icon(Icons.Default.Settings, null) },
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

## プログラムで制御

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ControlledSheet() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()
    val scope = rememberCoroutineScope()

    Button(onClick = { showSheet = true }) { Text("開く") }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("コンテンツ")
                Spacer(Modifier.height(16.dp))
                Button(onClick = {
                    scope.launch {
                        sheetState.hide()
                        showSheet = false
                    }
                }) { Text("閉じる") }
            }
        }
    }
}
```

---

## リスト付きBottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FilterBottomSheet(
    filters: List<String>,
    selected: Set<String>,
    onToggle: (String) -> Unit,
    onDismiss: () -> Unit
) {
    ModalBottomSheet(onDismissRequest = onDismiss) {
        Text(
            "フィルター",
            style = MaterialTheme.typography.titleLarge,
            modifier = Modifier.padding(horizontal = 16.dp)
        )
        Spacer(Modifier.height(8.dp))
        LazyColumn {
            items(filters) { filter ->
                ListItem(
                    headlineContent = { Text(filter) },
                    trailingContent = {
                        Checkbox(
                            checked = filter in selected,
                            onCheckedChange = { onToggle(filter) }
                        )
                    },
                    modifier = Modifier.clickable { onToggle(filter) }
                )
            }
        }
        Spacer(Modifier.height(32.dp))
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ModalBottomSheet` | モーダルシート表示 |
| `SheetState` | 展開状態制御 |
| `skipPartiallyExpanded` | 半展開スキップ |
| `rememberModalBottomSheetState` | 状態保持 |

- `ModalBottomSheet`でMaterial3準拠のシート
- `SheetState`でプログラムから展開/非表示制御
- `skipPartiallyExpanded`で全画面展開のみに
- `onDismissRequest`でスワイプ/バック閉じ対応

---

8種類のAndroidアプリテンプレート（UI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [Dialog](https://zenn.dev/myougatheaxo/articles/android-compose-dialog-2026)
- [Popup Menu](https://zenn.dev/myougatheaxo/articles/android-compose-popup-menu-2026)
