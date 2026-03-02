---
title: "BottomNavigation完全ガイド — NavigationBar/バッジ/FAB連携/アニメーション"
emoji: "🧭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**BottomNavigation**（NavigationBar、バッジ表示、FAB連携、スクロール時の非表示アニメーション）を解説します。

---

## NavigationBar

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val currentRoute by navController.currentBackStackEntryAsState()

    Scaffold(
        bottomBar = {
            NavigationBar {
                bottomNavItems.forEach { item ->
                    NavigationBarItem(
                        icon = {
                            if (item.badgeCount > 0) {
                                BadgedBox(badge = { Badge { Text("${item.badgeCount}") } }) {
                                    Icon(item.icon, contentDescription = item.label)
                                }
                            } else {
                                Icon(item.icon, contentDescription = item.label)
                            }
                        },
                        label = { Text(item.label) },
                        selected = currentRoute?.destination?.route == item.route,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.findStartDestination().id) {
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
        NavHost(navController, startDestination = "home", Modifier.padding(padding)) {
            composable("home") { HomeScreen() }
            composable("search") { SearchScreen() }
            composable("profile") { ProfileScreen() }
        }
    }
}

data class BottomNavItem(
    val route: String,
    val label: String,
    val icon: ImageVector,
    val badgeCount: Int = 0
)

val bottomNavItems = listOf(
    BottomNavItem("home", "ホーム", Icons.Default.Home),
    BottomNavItem("search", "検索", Icons.Default.Search),
    BottomNavItem("profile", "プロフィール", Icons.Default.Person, badgeCount = 3)
)
```

---

## スクロール時非表示

```kotlin
@Composable
fun HideOnScrollBottomBar(content: @Composable (PaddingValues) -> Unit) {
    val scrollBehavior = BottomAppBarDefaults.exitAlwaysScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        bottomBar = {
            BottomAppBar(scrollBehavior = scrollBehavior) {
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Home, null) },
                    label = { Text("ホーム") },
                    selected = true,
                    onClick = {}
                )
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Settings, null) },
                    label = { Text("設定") },
                    selected = false,
                    onClick = {}
                )
            }
        }
    ) { padding -> content(padding) }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| ナビバー | `NavigationBar` |
| アイテム | `NavigationBarItem` |
| バッジ | `BadgedBox` + `Badge` |
| スクロール連動 | `exitAlwaysScrollBehavior` |

- `NavigationBar`でMaterial3ボトムナビゲーション
- `BadgedBox`で未読バッジ表示
- `popUpTo`+`saveState`でバックスタック最適化
- `exitAlwaysScrollBehavior`でスクロール時に自動非表示

---

8種類のAndroidアプリテンプレート（ナビゲーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [NavigationDrawer](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-drawer-2026)
- [TabRow/Pager](https://zenn.dev/myougatheaxo/articles/android-compose-tabrow-viewpager-2026)
