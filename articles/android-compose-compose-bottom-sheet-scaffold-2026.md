---
title: "Compose BottomSheetScaffold完全ガイド — 固定BottomSheet/Peek/スワイプ制御"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose BottomSheetScaffold**（固定BottomSheet、peekHeight、スワイプ制御、コンテンツ連動）を解説します。

---

## 基本BottomSheetScaffold

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BottomSheetScaffoldDemo() {
    val scaffoldState = rememberBottomSheetScaffoldState()

    BottomSheetScaffold(
        scaffoldState = scaffoldState,
        sheetPeekHeight = 100.dp,
        sheetContent = {
            Column(Modifier.fillMaxWidth().padding(16.dp)) {
                Text("シートコンテンツ", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(8.dp))
                repeat(10) {
                    ListItem(headlineContent = { Text("アイテム $it") })
                }
            }
        },
        topBar = { TopAppBar(title = { Text("BottomSheetScaffold") }) }
    ) { padding ->
        Box(Modifier.fillMaxSize().padding(padding), contentAlignment = Alignment.Center) {
            Text("メインコンテンツ")
        }
    }
}
```

---

## プログラムによる制御

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ControlledBottomSheet() {
    val scaffoldState = rememberBottomSheetScaffoldState(
        bottomSheetState = rememberStandardBottomSheetState(
            initialValue = SheetValue.PartiallyExpanded
        )
    )
    val scope = rememberCoroutineScope()

    BottomSheetScaffold(
        scaffoldState = scaffoldState,
        sheetPeekHeight = 80.dp,
        sheetContent = {
            Column(Modifier.padding(16.dp)) {
                Text("詳細情報", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(16.dp))
                Text("ここにフィルタや詳細情報を表示")
                repeat(5) {
                    ListItem(headlineContent = { Text("オプション $it") })
                }
            }
        }
    ) { padding ->
        Column(Modifier.padding(padding).padding(16.dp)) {
            Button(onClick = {
                scope.launch { scaffoldState.bottomSheetState.expand() }
            }) { Text("シートを展開") }

            Spacer(Modifier.height(8.dp))

            OutlinedButton(onClick = {
                scope.launch { scaffoldState.bottomSheetState.partialExpand() }
            }) { Text("シートを縮小") }
        }
    }
}
```

---

## 地図+BottomSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MapWithBottomSheet() {
    val scaffoldState = rememberBottomSheetScaffoldState()

    BottomSheetScaffold(
        scaffoldState = scaffoldState,
        sheetPeekHeight = 120.dp,
        sheetContent = {
            Column(Modifier.fillMaxWidth().padding(16.dp)) {
                // ハンドル
                Box(Modifier.width(40.dp).height(4.dp)
                    .background(Color.Gray, RoundedCornerShape(2.dp))
                    .align(Alignment.CenterHorizontally))
                Spacer(Modifier.height(12.dp))
                Text("近くのスポット", style = MaterialTheme.typography.titleMedium)
                repeat(10) { index ->
                    ListItem(
                        headlineContent = { Text("スポット $index") },
                        supportingContent = { Text("${(index + 1) * 100}m") },
                        leadingContent = { Icon(Icons.Default.Place, null) }
                    )
                }
            }
        }
    ) { padding ->
        // 地図表示エリア
        Box(Modifier.fillMaxSize().padding(padding)
            .background(Color(0xFFE8F5E9))) {
            Text("地図", Modifier.align(Alignment.Center),
                style = MaterialTheme.typography.displayLarge)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BottomSheetScaffold` | 固定BottomSheet |
| `sheetPeekHeight` | 覗き見の高さ |
| `expand()` | 完全展開 |
| `partialExpand()` | 部分展開 |

- `BottomSheetScaffold`は常に画面下に表示される固定シート
- `ModalBottomSheet`はオーバーレイ型（閉じられる）
- `sheetPeekHeight`で部分表示時の高さを設定
- 地図アプリのスポット一覧など「覗き見→展開」パターンに最適

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ModalBottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-modal-bottom-sheet-2026)
- [Compose Scaffold+Snackbar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-snackbar-2026)
- [Compose GoogleMap](https://zenn.dev/myougatheaxo/articles/android-compose-compose-google-map-2026)
