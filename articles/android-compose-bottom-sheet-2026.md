---
title: "BottomSheet完全ガイド — Composeでモーダル・標準シートを実装"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**ModalBottomSheet**と**BottomSheetScaffold**の実装方法を解説します。

---

## ModalBottomSheet（モーダル）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ModalBottomSheetExample() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()

    Button(onClick = { showSheet = true }) {
        Text("シートを開く")
    }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("メニュー", style = MaterialTheme.typography.titleLarge)
                Spacer(Modifier.height(16.dp))

                ListItem(
                    headlineContent = { Text("共有") },
                    leadingContent = { Icon(Icons.Default.Share, null) },
                    modifier = Modifier.clickable { showSheet = false }
                )
                ListItem(
                    headlineContent = { Text("リンクをコピー") },
                    leadingContent = { Icon(Icons.Default.ContentCopy, null) },
                    modifier = Modifier.clickable { showSheet = false }
                )
                ListItem(
                    headlineContent = { Text("削除") },
                    leadingContent = { Icon(Icons.Default.Delete, null) },
                    colors = ListItemDefaults.colors(
                        headlineColor = MaterialTheme.colorScheme.error,
                        leadingIconColor = MaterialTheme.colorScheme.error
                    ),
                    modifier = Modifier.clickable { showSheet = false }
                )

                Spacer(Modifier.height(32.dp))
            }
        }
    }
}
```

---

## skipPartiallyExpanded

```kotlin
val sheetState = rememberModalBottomSheetState(
    skipPartiallyExpanded = true  // 半開き状態をスキップ
)
```

- `false`（デフォルト）: 半開き → 全開きの2段階
- `true`: 最初から全開き

---

## BottomSheetScaffold（常駐シート）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BottomSheetScaffoldExample() {
    val scaffoldState = rememberBottomSheetScaffoldState()

    BottomSheetScaffold(
        scaffoldState = scaffoldState,
        sheetContent = {
            Column(Modifier.padding(16.dp)) {
                Text("詳細情報", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(8.dp))
                Text("ここにシートの内容を配置します。")
                Text("上にスワイプで全画面表示。")
                Spacer(Modifier.height(200.dp))
                Text("追加の情報...")
            }
        },
        sheetPeekHeight = 128.dp,
        sheetShape = RoundedCornerShape(topStart = 16.dp, topEnd = 16.dp)
    ) { padding ->
        // メインコンテンツ
        LazyColumn(contentPadding = padding) {
            items(50) { index ->
                ListItem(headlineContent = { Text("アイテム $index") })
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
    val scope = rememberCoroutineScope()
    val sheetState = rememberModalBottomSheetState()
    var showSheet by remember { mutableStateOf(false) }

    // 閉じる
    Button(onClick = {
        scope.launch { sheetState.hide() }.invokeOnCompletion {
            if (!sheetState.isVisible) showSheet = false
        }
    }) {
        Text("閉じる")
    }

    // 全開き
    Button(onClick = {
        scope.launch { sheetState.expand() }
    }) {
        Text("全開き")
    }
}
```

---

## フォーム付きBottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FormBottomSheet(
    onDismiss: () -> Unit,
    onSubmit: (String, String) -> Unit
) {
    var title by remember { mutableStateOf("") }
    var description by remember { mutableStateOf("") }

    ModalBottomSheet(onDismissRequest = onDismiss) {
        Column(
            Modifier
                .padding(horizontal = 16.dp)
                .imePadding()  // キーボード対応
        ) {
            Text("新規タスク", style = MaterialTheme.typography.titleLarge)
            Spacer(Modifier.height(16.dp))

            OutlinedTextField(
                value = title,
                onValueChange = { title = it },
                label = { Text("タイトル") },
                modifier = Modifier.fillMaxWidth()
            )
            Spacer(Modifier.height(8.dp))

            OutlinedTextField(
                value = description,
                onValueChange = { description = it },
                label = { Text("説明") },
                modifier = Modifier.fillMaxWidth(),
                minLines = 3
            )
            Spacer(Modifier.height(16.dp))

            Button(
                onClick = { onSubmit(title, description) },
                modifier = Modifier.fillMaxWidth(),
                enabled = title.isNotBlank()
            ) {
                Text("追加")
            }
            Spacer(Modifier.height(32.dp))
        }
    }
}
```

---

## まとめ

- `ModalBottomSheet` — オーバーレイ表示のモーダルシート
- `BottomSheetScaffold` — 常駐型の下部シート
- `rememberModalBottomSheetState()` で状態管理
- `skipPartiallyExpanded` で半開きスキップ
- `imePadding()` でキーボード対応

---

8種類のAndroidアプリテンプレート（BottomSheet設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ダイアログ完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-dialog-guide-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
