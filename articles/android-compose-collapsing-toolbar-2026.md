---
title: "CollapsingToolbar完全ガイド — 折りたたみAppBar/スクロール連動/パララックス"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**CollapsingToolbar**（折りたたみAppBar、スクロール連動、パララックスヘッダー、Scaffold統合）を解説します。

---

## TopAppBarScrollBehavior

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CollapsingToolbarScreen() {
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            LargeTopAppBar(
                title = { Text("マイアプリ") },
                navigationIcon = {
                    IconButton(onClick = { }) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, "戻る")
                    }
                },
                actions = {
                    IconButton(onClick = { }) { Icon(Icons.Default.Search, "検索") }
                    IconButton(onClick = { }) { Icon(Icons.Default.MoreVert, "メニュー") }
                },
                scrollBehavior = scrollBehavior
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(50) { index ->
                ListItem(headlineContent = { Text("Item $index") })
            }
        }
    }
}
```

---

## MediumTopAppBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MediumToolbarScreen() {
    val scrollBehavior = TopAppBarDefaults.enterAlwaysScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            MediumTopAppBar(
                title = { Text("設定") },
                scrollBehavior = scrollBehavior,
                colors = TopAppBarDefaults.mediumTopAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    scrolledContainerColor = MaterialTheme.colorScheme.surface
                )
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(30) { ListItem(headlineContent = { Text("Setting $it") }) }
        }
    }
}
```

---

## カスタムCollapsingHeader

```kotlin
@Composable
fun CustomCollapsingHeader() {
    val lazyListState = rememberLazyListState()
    val headerHeight = 250.dp
    val minHeaderHeight = 64.dp
    val headerHeightPx = with(LocalDensity.current) { headerHeight.toPx() }
    val minHeaderHeightPx = with(LocalDensity.current) { minHeaderHeight.toPx() }

    Box(Modifier.fillMaxSize()) {
        LazyColumn(state = lazyListState, contentPadding = PaddingValues(top = headerHeight)) {
            items(50) { index ->
                ListItem(headlineContent = { Text("Item $index") })
            }
        }

        // 折りたたみヘッダー
        val scrollOffset = lazyListState.firstVisibleItemScrollOffset.toFloat()
        val firstIndex = lazyListState.firstVisibleItemIndex
        val collapse = if (firstIndex == 0) {
            (scrollOffset / (headerHeightPx - minHeaderHeightPx)).coerceIn(0f, 1f)
        } else 1f

        val currentHeight = lerp(headerHeight, minHeaderHeight, collapse)

        Surface(
            Modifier.fillMaxWidth().height(currentHeight),
            color = MaterialTheme.colorScheme.primaryContainer,
            shadowElevation = if (collapse > 0.9f) 4.dp else 0.dp
        ) {
            Box(contentAlignment = Alignment.BottomStart) {
                // 背景画像（パララックス）
                Image(
                    painter = painterResource(R.drawable.header),
                    contentDescription = null,
                    modifier = Modifier
                        .fillMaxSize()
                        .graphicsLayer { alpha = 1f - collapse },
                    contentScale = ContentScale.Crop
                )

                Text(
                    "プロフィール",
                    modifier = Modifier.padding(16.dp),
                    style = MaterialTheme.typography.headlineMedium.copy(
                        fontSize = lerp(28.sp, 20.sp, collapse)
                    )
                )
            }
        }
    }
}
```

---

## ScrollBehavior比較

```kotlin
// exitUntilCollapsed: スクロールダウンで縮小、上端まで戻ると展開
// enterAlways: スクロールアップですぐ展開
// pinnedScrollBehavior: 常に表示、色のみ変化

@OptIn(ExperimentalMaterial3Api::class)
val behaviors = mapOf(
    "exitUntilCollapsed" to TopAppBarDefaults.exitUntilCollapsedScrollBehavior(),
    "enterAlways" to TopAppBarDefaults.enterAlwaysScrollBehavior(),
    "pinned" to TopAppBarDefaults.pinnedScrollBehavior()
)
```

---

## まとめ

| Behavior | 動作 |
|----------|------|
| `exitUntilCollapsed` | スクロールで縮小、上端で展開 |
| `enterAlways` | 上スクロールで即展開 |
| `pinned` | 常に固定表示 |
| カスタム | 自由なアニメーション |

- `LargeTopAppBar` + `exitUntilCollapsed`で標準的な折りたたみ
- `nestedScroll`でスクロール連動を接続
- カスタム実装でパララックスヘッダー
- `lerp`でサイズ/フォントサイズをスムーズ補間

---

8種類のAndroidアプリテンプレート（CollapsingToolbar実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold/TopBar](https://zenn.dev/myougatheaxo/articles/android-compose-scaffold-topbar-2026)
- [パララックス](https://zenn.dev/myougatheaxo/articles/android-compose-parallax-scroll-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-optimization-2026)
