---
title: "Compose BottomNavigation完全ガイド — タブ切り替えの正しい実装"
emoji: "🗂️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

Composeで**BottomNavigation（タブ切り替え）**を正しく実装する方法を解説します。バックスタック管理やバッジ表示も含みます。

---

## 基本実装

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()

    Scaffold(
        bottomBar = { BottomNavigationBar(navController) }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(padding)
        ) {
            composable("home") { HomeScreen() }
            composable("search") { SearchScreen() }
            composable("profile") { ProfileScreen() }
        }
    }
}
```

---

## NavigationBar（Material3）

```kotlin
@Composable
fun BottomNavigationBar(navController: NavController) {
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route

    NavigationBar {
        BottomNavItem.entries.forEach { item ->
            NavigationBarItem(
                icon = {
                    Icon(
                        if (currentRoute == item.route) item.selectedIcon
                        else item.unselectedIcon,
                        contentDescription = item.label
                    )
                },
                label = { Text(item.label) },
                selected = currentRoute == item.route,
                onClick = {
                    navController.navigate(item.route) {
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

enum class BottomNavItem(
    val route: String,
    val label: String,
    val selectedIcon: ImageVector,
    val unselectedIcon: ImageVector
) {
    HOME("home", "ホーム", Icons.Filled.Home, Icons.Outlined.Home),
    SEARCH("search", "検索", Icons.Filled.Search, Icons.Outlined.Search),
    PROFILE("profile", "プロフィール", Icons.Filled.Person, Icons.Outlined.Person)
}
```

---

## バックスタックの管理

```kotlin
navController.navigate(item.route) {
    // スタートデスティネーションまでポップ（重複防止）
    popUpTo(navController.graph.startDestinationId) {
        saveState = true      // 前のタブの状態を保存
    }
    launchSingleTop = true    // 同じタブの重複防止
    restoreState = true       // タブ切り替え時に状態を復元
}
```

これにより：
- タブ切り替えでスタックが積み重ならない
- 前のタブに戻ったとき状態が復元される
- 同じタブを連打しても重複しない

---

## バッジ表示

```kotlin
NavigationBarItem(
    icon = {
        BadgedBox(
            badge = {
                if (item == BottomNavItem.HOME && unreadCount > 0) {
                    Badge { Text("$unreadCount") }
                }
            }
        ) {
            Icon(item.selectedIcon, item.label)
        }
    },
    label = { Text(item.label) },
    selected = currentRoute == item.route,
    onClick = { /* ... */ }
)
```

---

## タブ内のネスト画面

```kotlin
NavHost(navController, startDestination = "home") {
    // ホームタブ
    navigation(startDestination = "home/list", route = "home") {
        composable("home/list") {
            HomeListScreen(
                onItemClick = { id ->
                    navController.navigate("home/detail/$id")
                }
            )
        }
        composable("home/detail/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: ""
            HomeDetailScreen(id)
        }
    }

    // 検索タブ
    composable("search") { SearchScreen() }

    // プロフィールタブ
    composable("profile") { ProfileScreen() }
}
```

---

## BottomNavの表示/非表示

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()

    // 詳細画面ではBottomNavを非表示
    val showBottomBar = when (navBackStackEntry?.destination?.route) {
        "home/detail/{id}" -> false
        else -> true
    }

    Scaffold(
        bottomBar = {
            AnimatedVisibility(visible = showBottomBar) {
                BottomNavigationBar(navController)
            }
        }
    ) { padding ->
        NavHost(navController, "home", Modifier.padding(padding)) {
            // ...
        }
    }
}
```

---

## まとめ

- `NavigationBar` + `NavigationBarItem`でMaterial3準拠
- `popUpTo` + `saveState` + `restoreState`でバックスタック管理
- `BadgedBox`で未読バッジ表示
- `navigation()`でタブ内ネスト画面
- `AnimatedVisibility`で詳細画面時に非表示

---

8種類のAndroidアプリテンプレート（BottomNavigation実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [Navigation引数パターン集](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-args-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
