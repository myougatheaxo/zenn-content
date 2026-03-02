---
title: "Scaffold完全ガイド — TopAppBar/BottomBar/FAB/Snackbar/Drawer統合"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Scaffold**（TopAppBar、BottomBar、FAB、Snackbar、Drawer統合レイアウト）を解説します。

---

## 基本Scaffold

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MainScaffold() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scrollBehavior = TopAppBarDefaults.enterAlwaysScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            TopAppBar(
                title = { Text("マイアプリ") },
                navigationIcon = {
                    IconButton(onClick = {}) { Icon(Icons.Default.Menu, "メニュー") }
                },
                actions = {
                    IconButton(onClick = {}) { Icon(Icons.Default.Search, "検索") }
                    IconButton(onClick = {}) { Icon(Icons.Default.MoreVert, "その他") }
                },
                scrollBehavior = scrollBehavior
            )
        },
        bottomBar = {
            NavigationBar {
                NavigationBarItem(icon = { Icon(Icons.Default.Home, null) }, label = { Text("ホーム") }, selected = true, onClick = {})
                NavigationBarItem(icon = { Icon(Icons.Default.Person, null) }, label = { Text("プロフィール") }, selected = false, onClick = {})
            }
        },
        floatingActionButton = {
            FloatingActionButton(onClick = {}) {
                Icon(Icons.Default.Add, "追加")
            }
        },
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(50) { index ->
                ListItem(headlineContent = { Text("アイテム $index") })
            }
        }
    }
}
```

---

## LargeTopAppBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CollapsingAppBarScreen() {
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            LargeTopAppBar(
                title = { Text("設定") },
                navigationIcon = {
                    IconButton(onClick = {}) { Icon(Icons.AutoMirrored.Filled.ArrowBack, "戻る") }
                },
                scrollBehavior = scrollBehavior
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(30) { ListItem(headlineContent = { Text("設定項目 $it") }) }
        }
    }
}
```

---

## ModalDrawer統合

```kotlin
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
                NavigationDrawerItem(label = { Text("ホーム") }, selected = true, onClick = {})
                NavigationDrawerItem(label = { Text("設定") }, selected = false, onClick = {})
            }
        }
    ) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text("アプリ") },
                    navigationIcon = {
                        IconButton(onClick = { scope.launch { drawerState.open() } }) {
                            Icon(Icons.Default.Menu, "メニュー")
                        }
                    }
                )
            }
        ) { padding ->
            Box(Modifier.padding(padding)) { Text("コンテンツ") }
        }
    }
}
```

---

## まとめ

| スロット | コンポーネント |
|---------|-------------|
| topBar | TopAppBar / LargeTopAppBar |
| bottomBar | NavigationBar |
| floatingActionButton | FAB |
| snackbarHost | SnackbarHost |

- `Scaffold`でMaterial3準拠のレイアウト構造
- `scrollBehavior`でスクロール連動TopAppBar
- `ModalNavigationDrawer`でサイドメニュー
- `padding`パラメータで適切な余白管理

---

8種類のAndroidアプリテンプレート（Scaffold設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
- [NavigationDrawer](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-drawer-2026)
- [CollapsingToolbar](https://zenn.dev/myougatheaxo/articles/android-compose-collapsing-toolbar-2026)
