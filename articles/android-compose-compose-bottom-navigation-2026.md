---
title: "BottomNavigation完全ガイド — NavigationBar/バッジ/NavHost連携"
emoji: "🧭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**BottomNavigation**（NavigationBar、NavigationBarItem、バッジ、NavHost連携）を解説します。

---

## NavigationBar基本

```kotlin
@Composable
fun BottomNavScreen() {
    var selectedIndex by remember { mutableIntStateOf(0) }
    val items = listOf(
        Triple("ホーム", Icons.Default.Home, Icons.Outlined.Home),
        Triple("検索", Icons.Default.Search, Icons.Outlined.Search),
        Triple("通知", Icons.Default.Notifications, Icons.Outlined.Notifications),
        Triple("設定", Icons.Default.Settings, Icons.Outlined.Settings)
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                items.forEachIndexed { index, (label, selectedIcon, unselectedIcon) ->
                    NavigationBarItem(
                        selected = selectedIndex == index,
                        onClick = { selectedIndex = index },
                        icon = { Icon(if (selectedIndex == index) selectedIcon else unselectedIcon, label) },
                        label = { Text(label) }
                    )
                }
            }
        }
    ) { innerPadding ->
        Box(Modifier.padding(innerPadding).fillMaxSize(), contentAlignment = Alignment.Center) {
            Text("${items[selectedIndex].first}画面", style = MaterialTheme.typography.headlineMedium)
        }
    }
}
```

---

## バッジ付き

```kotlin
@Composable
fun BadgeBottomNav() {
    var selectedIndex by remember { mutableIntStateOf(0) }
    val notificationCount = 3

    Scaffold(
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    selected = selectedIndex == 0,
                    onClick = { selectedIndex = 0 },
                    icon = { Icon(Icons.Default.Home, null) },
                    label = { Text("ホーム") }
                )
                NavigationBarItem(
                    selected = selectedIndex == 1,
                    onClick = { selectedIndex = 1 },
                    icon = {
                        BadgedBox(badge = {
                            if (notificationCount > 0) Badge { Text("$notificationCount") }
                        }) { Icon(Icons.Default.Notifications, null) }
                    },
                    label = { Text("通知") }
                )
                NavigationBarItem(
                    selected = selectedIndex == 2,
                    onClick = { selectedIndex = 2 },
                    icon = {
                        BadgedBox(badge = { Badge() }) {
                            Icon(Icons.Default.Email, null)
                        }
                    },
                    label = { Text("メール") }
                )
            }
        }
    ) { Box(Modifier.padding(it)) }
}
```

---

## NavHost連携

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val currentRoute = navController.currentBackStackEntryAsState().value?.destination?.route

    Scaffold(
        bottomBar = {
            NavigationBar {
                listOf(
                    Triple("home", "ホーム", Icons.Default.Home),
                    Triple("search", "検索", Icons.Default.Search),
                    Triple("profile", "プロフィール", Icons.Default.Person)
                ).forEach { (route, label, icon) ->
                    NavigationBarItem(
                        selected = currentRoute == route,
                        onClick = {
                            navController.navigate(route) {
                                popUpTo(navController.graph.startDestinationId) { saveState = true }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        icon = { Icon(icon, label) },
                        label = { Text(label) }
                    )
                }
            }
        }
    ) { innerPadding ->
        NavHost(navController, "home", Modifier.padding(innerPadding)) {
            composable("home") { Text("ホーム画面") }
            composable("search") { Text("検索画面") }
            composable("profile") { Text("プロフィール画面") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `NavigationBar` | ボトムナビゲーション |
| `NavigationBarItem` | 各タブアイテム |
| `BadgedBox` | バッジ表示 |
| `NavHost` | 画面遷移連携 |

- `NavigationBar`でMaterial3準拠のボトムナビ
- `BadgedBox`で通知バッジ表示
- `popUpTo` + `launchSingleTop`でバックスタック管理
- `restoreState`でタブの状態を保持

---

8種類のAndroidアプリテンプレート（ナビゲーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [NavigationRail](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-rail-2026)
- [NavigationDrawer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-drawer-2026)
- [Type-safe Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-type-safe-2026)
