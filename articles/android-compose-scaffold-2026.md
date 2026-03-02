---
title: "Scaffold完全ガイド — TopBar/BottomBar/FAB/Drawerの統合レイアウト"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "scaffold"]
published: true
---

## この記事で学べること

Composeの**Scaffold**を使ったアプリの骨格レイアウト（TopBar、BottomBar、FAB、Drawer）を解説します。

---

## 基本のScaffold

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BasicScaffold() {
    val snackbarHostState = remember { SnackbarHostState() }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("マイアプリ") },
                navigationIcon = {
                    IconButton(onClick = { /* メニュー */ }) {
                        Icon(Icons.Default.Menu, "メニュー")
                    }
                },
                actions = {
                    IconButton(onClick = { /* 検索 */ }) {
                        Icon(Icons.Default.Search, "検索")
                    }
                    IconButton(onClick = { /* 設定 */ }) {
                        Icon(Icons.Default.Settings, "設定")
                    }
                }
            )
        },
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Home, "ホーム") },
                    label = { Text("ホーム") },
                    selected = true,
                    onClick = {}
                )
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Person, "プロフィール") },
                    label = { Text("プロフィール") },
                    selected = false,
                    onClick = {}
                )
            }
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* 追加 */ }) {
                Icon(Icons.Default.Add, "追加")
            }
        },
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        // メインコンテンツ（paddingを適用）
        LazyColumn(
            modifier = Modifier.padding(padding)
        ) {
            items(50) {
                ListItem(headlineContent = { Text("Item $it") })
            }
        }
    }
}
```

---

## スクロール連動TopAppBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CollapsingTopBarScaffold() {
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            LargeTopAppBar(
                title = { Text("設定") },
                scrollBehavior = scrollBehavior,
                navigationIcon = {
                    IconButton(onClick = { }) {
                        Icon(Icons.AutoMirrored.Default.ArrowBack, "戻る")
                    }
                }
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(30) {
                ListItem(
                    headlineContent = { Text("設定項目 $it") },
                    trailingContent = { Switch(checked = false, onCheckedChange = {}) }
                )
            }
        }
    }
}
```

---

## 詳細画面Scaffold

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DetailScaffold(
    title: String,
    onBack: () -> Unit,
    onShare: () -> Unit,
    onDelete: () -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(title, maxLines = 1, overflow = TextOverflow.Ellipsis) },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.AutoMirrored.Default.ArrowBack, "戻る")
                    }
                },
                actions = {
                    IconButton(onClick = onShare) {
                        Icon(Icons.Default.Share, "共有")
                    }
                    IconButton(onClick = onDelete) {
                        Icon(Icons.Default.Delete, "削除")
                    }
                }
            )
        }
    ) { padding ->
        Column(
            Modifier.padding(padding).verticalScroll(rememberScrollState()).padding(16.dp)
        ) {
            Text("詳細コンテンツ", style = MaterialTheme.typography.bodyLarge)
        }
    }
}
```

---

## ModalNavigationDrawer + Scaffold

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DrawerScaffold() {
    val drawerState = rememberDrawerState(DrawerValue.Closed)
    val scope = rememberCoroutineScope()

    ModalNavigationDrawer(
        drawerState = drawerState,
        drawerContent = {
            ModalDrawerSheet {
                Text("メニュー", Modifier.padding(16.dp), style = MaterialTheme.typography.headlineSmall)
                HorizontalDivider()
                NavigationDrawerItem(
                    label = { Text("ホーム") },
                    icon = { Icon(Icons.Default.Home, null) },
                    selected = true,
                    onClick = { scope.launch { drawerState.close() } }
                )
                NavigationDrawerItem(
                    label = { Text("設定") },
                    icon = { Icon(Icons.Default.Settings, null) },
                    selected = false,
                    onClick = { scope.launch { drawerState.close() } }
                )
            }
        }
    ) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text("マイアプリ") },
                    navigationIcon = {
                        IconButton(onClick = { scope.launch { drawerState.open() } }) {
                            Icon(Icons.Default.Menu, "メニュー")
                        }
                    }
                )
            }
        ) { padding ->
            Box(Modifier.padding(padding)) {
                Text("メインコンテンツ")
            }
        }
    }
}
```

---

## まとめ

- `Scaffold`でTopBar/BottomBar/FAB/Snackbar/Drawerを統合
- `padding`パラメータを必ずコンテンツに適用
- `LargeTopAppBar` + `scrollBehavior`でスクロール連動
- `ModalNavigationDrawer`はScaffoldの外側に配置
- `snackbarHost`でSnackbar表示
- `nestedScroll`でスクロールとTopBarの連動

---

8種類のAndroidアプリテンプレート（Scaffold設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [TopAppBarガイド](https://zenn.dev/myougatheaxo/articles/android-compose-app-bar-2026)
- [NavigationDrawerガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-drawer-2026)
- [FAB実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-floating-action-button-2026)
