---
title: "Compose ScrollState完全ガイド — スクロール検出/FAB表示切替/トップへ戻る"
emoji: "📜"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose ScrollState**（スクロール位置取得、LazyListState、FAB表示切替、トップへ戻るボタン）を解説します。

---

## 基本ScrollState

```kotlin
@Composable
fun ScrollStateDemo() {
    val scrollState = rememberScrollState()

    Column(Modifier.verticalScroll(scrollState).padding(16.dp)) {
        Text("スクロール位置: ${scrollState.value}px")
        Text("最大値: ${scrollState.maxValue}px")

        repeat(50) {
            Text("アイテム $it", Modifier.padding(8.dp))
        }
    }
}
```

---

## LazyListStateでFAB制御

```kotlin
@Composable
fun ScrollAwareFabDemo() {
    val listState = rememberLazyListState()
    val scope = rememberCoroutineScope()

    // スクロール方向を検出
    val showFab by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 3 }
    }

    Scaffold(
        floatingActionButton = {
            AnimatedVisibility(visible = showFab, enter = fadeIn(), exit = fadeOut()) {
                FloatingActionButton(onClick = {
                    scope.launch { listState.animateScrollToItem(0) }
                }) { Icon(Icons.Default.KeyboardArrowUp, "トップへ") }
            }
        }
    ) { padding ->
        LazyColumn(state = listState, modifier = Modifier.padding(padding)) {
            items(100) { index ->
                ListItem(headlineContent = { Text("アイテム $index") })
                HorizontalDivider()
            }
        }
    }
}
```

---

## スクロール方向検出

```kotlin
@Composable
fun ScrollDirectionDemo() {
    val listState = rememberLazyListState()
    var isScrollingUp by remember { mutableStateOf(true) }

    // スクロール方向を検出
    LaunchedEffect(listState) {
        var previousIndex = listState.firstVisibleItemIndex
        var previousOffset = listState.firstVisibleItemScrollOffset
        snapshotFlow { listState.firstVisibleItemIndex to listState.firstVisibleItemScrollOffset }
            .collect { (index, offset) ->
                isScrollingUp = if (index != previousIndex) {
                    index < previousIndex
                } else {
                    offset < previousOffset
                }
                previousIndex = index
                previousOffset = offset
            }
    }

    Scaffold(
        topBar = {
            AnimatedVisibility(visible = isScrollingUp, enter = slideInVertically(),
                exit = slideOutVertically()) {
                TopAppBar(title = { Text("スクロールで表示/非表示") })
            }
        }
    ) { padding ->
        LazyColumn(state = listState, modifier = Modifier.padding(padding)) {
            items(100) { Text("アイテム $it", Modifier.padding(16.dp)) }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `rememberScrollState` | Column/Rowスクロール |
| `rememberLazyListState` | LazyColumn状態 |
| `animateScrollToItem` | スムーズスクロール |
| `derivedStateOf` | スクロール状態派生 |

- `ScrollState`でColumn/Rowのスクロール位置を取得
- `LazyListState`でLazyColumnの表示アイテム情報
- `derivedStateOf`で不要なリコンポジションを回避
- `snapshotFlow`でスクロール方向を検出

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose NestedScroll](https://zenn.dev/myougatheaxo/articles/android-compose-compose-nested-scroll-2026)
- [Compose PullRefresh](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pull-refresh-2026)
- [Compose LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-column-2026)
