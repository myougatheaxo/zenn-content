---
title: "BottomNavigation + Scaffold完全ガイド — Compose版"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

Composeの**BottomNavigation + Scaffold**（NavigationBar、バッジ、FAB、DrawerMenu）を解説します。

---

## NavigationBar基本

```kotlin
@Composable
fun MainScreen() {
    var selectedTab by remember { mutableIntStateOf(0) }

    Scaffold(
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    selected = selectedTab == 0,
                    onClick = { selectedTab = 0 },
                    icon = { Icon(Icons.Default.Home, "ホーム") },
                    label = { Text("ホーム") }
                )
                NavigationBarItem(
                    selected = selectedTab == 1,
                    onClick = { selectedTab = 1 },
                    icon = { Icon(Icons.Default.Search, "検索") },
                    label = { Text("検索") }
                )
                NavigationBarItem(
                    selected = selectedTab == 2,
                    onClick = { selectedTab = 2 },
                    icon = { Icon(Icons.Default.Person, "プロフィール") },
                    label = { Text("プロフィール") }
                )
            }
        }
    ) { padding ->
        Box(Modifier.padding(padding)) {
            when (selectedTab) {
                0 -> HomeScreen()
                1 -> SearchScreen()
                2 -> ProfileScreen()
            }
        }
    }
}
```

---

## Navigation連携

```kotlin
@Composable
fun AppWithBottomNav() {
    val navController = rememberNavController()
    val currentRoute by navController.currentBackStackEntryAsState()

    data class BottomNavItem(val route: String, val icon: ImageVector, val label: String)

    val items = listOf(
        BottomNavItem("home", Icons.Default.Home, "ホーム"),
        BottomNavItem("search", Icons.Default.Search, "検索"),
        BottomNavItem("profile", Icons.Default.Person, "プロフィール")
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                items.forEach { item ->
                    NavigationBarItem(
                        selected = currentRoute?.destination?.route == item.route,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        icon = { Icon(item.icon, item.label) },
                        label = { Text(item.label) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(navController, "home", Modifier.padding(padding)) {
            composable("home") { HomeScreen() }
            composable("search") { SearchScreen() }
            composable("profile") { ProfileScreen() }
        }
    }
}
```

---

## バッジ付きNavigationBar

```kotlin
@Composable
fun BadgedNavBar(unreadCount: Int) {
    NavigationBar {
        NavigationBarItem(
            selected = true,
            onClick = {},
            icon = {
                BadgedBox(
                    badge = {
                        if (unreadCount > 0) {
                            Badge { Text("$unreadCount") }
                        }
                    }
                ) {
                    Icon(Icons.Default.Notifications, "通知")
                }
            },
            label = { Text("通知") }
        )
    }
}
```

---

## FAB + Scaffold

```kotlin
@Composable
fun ScaffoldWithFab() {
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("タスク一覧") })
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* 追加 */ }) {
                Icon(Icons.Default.Add, "追加")
            }
        },
        floatingActionButtonPosition = FabPosition.End
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(10) { index ->
                ListItem(headlineContent = { Text("タスク $index") })
            }
        }
    }
}
```

---

## まとめ

- `NavigationBar`+`NavigationBarItem`でMaterial3ボトムナビ
- `NavHost`連携で`popUpTo`+`launchSingleTop`で正しいバックスタック
- `BadgedBox`で未読バッジ表示
- `FloatingActionButton`で主要アクション
- `TopAppBar`+`Scaffold`で統一レイアウト
- `currentBackStackEntryAsState`で現在タブの同期

---

8種類のAndroidアプリテンプレート（ナビゲーション実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [型安全ナビゲーション](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
- [タブUI実装](https://zenn.dev/myougatheaxo/articles/android-compose-tab-viewpager-2026)
- [バッジ/インジケーター](https://zenn.dev/myougatheaxo/articles/android-compose-badge-indicator-2026)
