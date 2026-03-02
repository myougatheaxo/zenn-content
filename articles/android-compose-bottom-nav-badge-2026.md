---
title: "BottomNavigation完全ガイド — バッジ付きナビゲーションバーの実装"
emoji: "📍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

Composeの**NavigationBar**（BottomNavigation）とバッジ表示の実装方法を解説します。

---

## 基本のNavigationBar

```kotlin
@Composable
fun MainScreen() {
    var selectedTab by rememberSaveable { mutableIntStateOf(0) }
    val navController = rememberNavController()

    val tabs = listOf(
        TabItem("ホーム", Icons.Default.Home, "home"),
        TabItem("検索", Icons.Default.Search, "search"),
        TabItem("通知", Icons.Default.Notifications, "notifications"),
        TabItem("設定", Icons.Default.Settings, "settings")
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                tabs.forEachIndexed { index, tab ->
                    NavigationBarItem(
                        icon = { Icon(tab.icon, tab.label) },
                        label = { Text(tab.label) },
                        selected = selectedTab == index,
                        onClick = {
                            selectedTab = index
                            navController.navigate(tab.route) {
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(padding)
        ) {
            composable("home") { HomeScreen() }
            composable("search") { SearchScreen() }
            composable("notifications") { NotificationScreen() }
            composable("settings") { SettingsScreen() }
        }
    }
}

data class TabItem(val label: String, val icon: ImageVector, val route: String)
```

---

## バッジ付きNavigationBar

```kotlin
@Composable
fun BadgedNavigationBar(notificationCount: Int) {
    var selectedTab by rememberSaveable { mutableIntStateOf(0) }

    NavigationBar {
        NavigationBarItem(
            icon = { Icon(Icons.Default.Home, "ホーム") },
            label = { Text("ホーム") },
            selected = selectedTab == 0,
            onClick = { selectedTab = 0 }
        )

        NavigationBarItem(
            icon = {
                BadgedBox(
                    badge = {
                        if (notificationCount > 0) {
                            Badge {
                                Text(
                                    if (notificationCount > 99) "99+"
                                    else "$notificationCount"
                                )
                            }
                        }
                    }
                ) {
                    Icon(Icons.Default.Notifications, "通知")
                }
            },
            label = { Text("通知") },
            selected = selectedTab == 1,
            onClick = { selectedTab = 1 }
        )

        NavigationBarItem(
            icon = {
                BadgedBox(badge = { Badge() }) { // ドット表示
                    Icon(Icons.Default.Email, "メール")
                }
            },
            label = { Text("メール") },
            selected = selectedTab == 2,
            onClick = { selectedTab = 2 }
        )
    }
}
```

---

## スクロール時の非表示

```kotlin
@Composable
fun HideableBottomNav() {
    val listState = rememberLazyListState()
    var isVisible by remember { mutableStateOf(true) }

    // スクロール方向を検出
    LaunchedEffect(listState) {
        var previousIndex = 0
        var previousOffset = 0
        snapshotFlow { listState.firstVisibleItemIndex to listState.firstVisibleItemScrollOffset }
            .collect { (index, offset) ->
                isVisible = if (index != previousIndex) {
                    index < previousIndex // 上スクロールで表示
                } else {
                    offset <= previousOffset
                }
                previousIndex = index
                previousOffset = offset
            }
    }

    Scaffold(
        bottomBar = {
            AnimatedVisibility(
                visible = isVisible,
                enter = slideInVertically(initialOffsetY = { it }),
                exit = slideOutVertically(targetOffsetY = { it })
            ) {
                NavigationBar {
                    // タブ項目
                }
            }
        }
    ) { padding ->
        LazyColumn(state = listState, modifier = Modifier.padding(padding)) {
            items(100) { Text("Item $it", Modifier.padding(16.dp)) }
        }
    }
}
```

---

## まとめ

- `NavigationBar` + `NavigationBarItem`でMaterial 3対応
- `navController.navigate()`で`popUpTo`/`launchSingleTop`/`restoreState`設定
- `BadgedBox` + `Badge`でバッジ表示
- `AnimatedVisibility`でスクロール時の表示/非表示
- `rememberSaveable`で選択状態を画面回転時も保持
- NavigationRailへの切り替えでタブレット対応

---

8種類のAndroidアプリテンプレート（ナビゲーション設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-basic-2026)
- [BadgeカウンターUI](https://zenn.dev/myougatheaxo/articles/android-compose-badge-counter-2026)
- [マルチスクリーン対応ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-multi-screen-2026)
