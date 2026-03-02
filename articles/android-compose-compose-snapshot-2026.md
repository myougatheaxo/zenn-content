---
title: "Snapshot完全ガイド — snapshotFlow/Snapshot API/状態監視"
emoji: "📷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

**Snapshot**（snapshotFlow、Snapshot API、Compose状態のFlow変換）を解説します。

---

## snapshotFlow

```kotlin
@Composable
fun SnapshotFlowExample() {
    val listState = rememberLazyListState()

    // Compose状態をFlowに変換
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                Analytics.logScrollPosition(index)
            }
    }

    LazyColumn(state = listState) {
        items(100) { ListItem(headlineContent = { Text("Item $it") }) }
    }
}
```

---

## スクロール方向検出

```kotlin
@Composable
fun ScrollDirectionDetector() {
    val listState = rememberLazyListState()
    var isScrollingUp by remember { mutableStateOf(false) }

    LaunchedEffect(listState) {
        var prevIndex = 0
        var prevOffset = 0

        snapshotFlow { listState.firstVisibleItemIndex to listState.firstVisibleItemScrollOffset }
            .collect { (index, offset) ->
                isScrollingUp = index < prevIndex || (index == prevIndex && offset < prevOffset)
                prevIndex = index
                prevOffset = offset
            }
    }

    Scaffold(
        topBar = {
            AnimatedVisibility(visible = isScrollingUp) {
                TopAppBar(title = { Text("ヘッダー") })
            }
        }
    ) { innerPadding ->
        LazyColumn(state = listState, contentPadding = innerPadding) {
            items(100) { ListItem(headlineContent = { Text("Item $it") }) }
        }
    }
}
```

---

## 状態変化のデバウンス

```kotlin
@Composable
fun DebouncedStateSync(viewModel: MyViewModel = hiltViewModel()) {
    var text by remember { mutableStateOf("") }

    LaunchedEffect(Unit) {
        snapshotFlow { text }
            .debounce(500)
            .distinctUntilChanged()
            .collect { viewModel.saveInput(it) }
    }

    OutlinedTextField(
        value = text, onValueChange = { text = it },
        label = { Text("自動保存") },
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `snapshotFlow` | Compose状態→Flow変換 |
| `distinctUntilChanged` | 重複フィルタ |
| `debounce` | 遅延処理 |
| Snapshot | 状態システムの基盤 |

- `snapshotFlow`でCompose状態をFlowオペレータで処理
- スクロール位置/入力値の監視に活用
- `distinctUntilChanged`で不要な処理を削減
- `debounce`でAPI呼び出し頻度制御

---

8種類のAndroidアプリテンプレート（状態管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [derivedStateOf](https://zenn.dev/myougatheaxo/articles/android-compose-compose-derived-state-2026)
- [Side Effect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effect-2026)
- [Flow debounce](https://zenn.dev/myougatheaxo/articles/android-compose-flow-debounce-2026)
