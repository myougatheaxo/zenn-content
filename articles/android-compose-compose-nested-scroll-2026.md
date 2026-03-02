---
title: "NestedScroll完全ガイド — nestedScroll/CollapsingToolbar/スクロール連携"
emoji: "📜"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "scroll"]
published: true
---

## この記事で学べること

**NestedScroll**（nestedScroll、CollapsingToolbar風、スクロール連携、消費と伝播）を解説します。

---

## Collapsing TopAppBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CollapsingToolbarExample() {
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            LargeTopAppBar(
                title = { Text("記事一覧") },
                scrollBehavior = scrollBehavior,
                navigationIcon = {
                    IconButton(onClick = {}) {
                        Icon(Icons.Default.ArrowBack, "戻る")
                    }
                }
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(50) { index ->
                ListItem(
                    headlineContent = { Text("記事 $index") },
                    supportingContent = { Text("Jetpack Composeの使い方") }
                )
            }
        }
    }
}
```

---

## カスタムNestedScroll

```kotlin
@Composable
fun CustomNestedScrollExample() {
    var headerHeight by remember { mutableFloatStateOf(200f) }
    val minHeight = 56f
    val maxHeight = 200f

    val nestedScrollConnection = remember {
        object : NestedScrollConnection {
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                val delta = available.y
                val newHeight = (headerHeight + delta).coerceIn(minHeight, maxHeight)
                val consumed = newHeight - headerHeight
                headerHeight = newHeight
                return Offset(0f, consumed)
            }
        }
    }

    Column(Modifier.nestedScroll(nestedScrollConnection)) {
        Box(
            Modifier
                .fillMaxWidth()
                .height(headerHeight.dp)
                .background(MaterialTheme.colorScheme.primaryContainer),
            contentAlignment = Alignment.Center
        ) {
            Text(
                "ヘッダー",
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier.graphicsLayer {
                    alpha = ((headerHeight - minHeight) / (maxHeight - minHeight))
                        .coerceIn(0f, 1f)
                }
            )
        }

        LazyColumn {
            items(30) { index ->
                ListItem(headlineContent = { Text("Item $index") })
            }
        }
    }
}
```

---

## スクロール方向検知

```kotlin
@Composable
fun ScrollDirectionDetector() {
    var isScrollingUp by remember { mutableStateOf(false) }

    val nestedScrollConnection = remember {
        object : NestedScrollConnection {
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                isScrollingUp = available.y > 0
                return Offset.Zero
            }
        }
    }

    Scaffold(
        modifier = Modifier.nestedScroll(nestedScrollConnection),
        floatingActionButton = {
            AnimatedVisibility(visible = isScrollingUp) {
                FloatingActionButton(onClick = {}) {
                    Icon(Icons.Default.Add, null)
                }
            }
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(50) { ListItem(headlineContent = { Text("Item $it") }) }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `nestedScroll` | スクロール連携 |
| `NestedScrollConnection` | スクロール消費 |
| `exitUntilCollapsedScrollBehavior` | Collapsing Toolbar |
| `onPreScroll` | スクロール前処理 |

- `nestedScroll`で親子のスクロール連携
- `NestedScrollConnection`でスクロール量を消費
- `LargeTopAppBar`+`scrollBehavior`でCollapsing Toolbar
- スクロール方向検知でFAB表示/非表示

---

8種類のAndroidアプリテンプレート（スクロール対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
- [PullToRefresh](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pull-refresh-2026)
