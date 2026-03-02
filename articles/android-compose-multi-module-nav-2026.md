---
title: "マルチモジュール + Navigation設計ガイド"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

マルチモジュールプロジェクトでの**Navigation設計パターン**を解説します。

---

## モジュール構成

```
app/                    ← NavHost定義
├── feature/
│   ├── home/           ← HomeScreen + HomeNavigation
│   ├── search/         ← SearchScreen + SearchNavigation
│   ├── profile/        ← ProfileScreen + ProfileNavigation
│   └── detail/         ← DetailScreen + DetailNavigation
├── core/
│   ├── ui/             ← 共通UIコンポーネント
│   ├── data/           ← Repository
│   ├── model/          ← ドメインモデル
│   └── navigation/     ← Navigation共通定義
```

---

## ルート定義（core:navigation）

```kotlin
// core/navigation
@Serializable
object HomeRoute

@Serializable
object SearchRoute

@Serializable
data class DetailRoute(val itemId: String)

@Serializable
object ProfileRoute
```

---

## Feature Navigationパターン

```kotlin
// feature/home/navigation/HomeNavigation.kt
fun NavGraphBuilder.homeScreen(
    onNavigateToDetail: (String) -> Unit,
    onNavigateToSearch: () -> Unit
) {
    composable<HomeRoute> {
        HomeScreen(
            onItemClick = onNavigateToDetail,
            onSearchClick = onNavigateToSearch
        )
    }
}

// feature/detail/navigation/DetailNavigation.kt
fun NavGraphBuilder.detailScreen(
    onBack: () -> Unit
) {
    composable<DetailRoute> { backStackEntry ->
        val route: DetailRoute = backStackEntry.toRoute()
        DetailScreen(
            itemId = route.itemId,
            onBack = onBack
        )
    }
}

// feature/search/navigation/SearchNavigation.kt
fun NavGraphBuilder.searchScreen(
    onNavigateToDetail: (String) -> Unit,
    onBack: () -> Unit
) {
    composable<SearchRoute> {
        SearchScreen(
            onItemClick = onNavigateToDetail,
            onBack = onBack
        )
    }
}
```

---

## App NavHost（app モジュール）

```kotlin
@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(
        navController = navController,
        startDestination = HomeRoute
    ) {
        homeScreen(
            onNavigateToDetail = { itemId ->
                navController.navigate(DetailRoute(itemId))
            },
            onNavigateToSearch = {
                navController.navigate(SearchRoute)
            }
        )

        searchScreen(
            onNavigateToDetail = { itemId ->
                navController.navigate(DetailRoute(itemId))
            },
            onBack = { navController.popBackStack() }
        )

        detailScreen(
            onBack = { navController.popBackStack() }
        )

        profileScreen(
            onNavigateToDetail = { itemId ->
                navController.navigate(DetailRoute(itemId))
            }
        )
    }
}
```

---

## BottomNavigation連携

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val items = listOf(
        BottomNavItem(HomeRoute, Icons.Default.Home, "ホーム"),
        BottomNavItem(SearchRoute, Icons.Default.Search, "検索"),
        BottomNavItem(ProfileRoute, Icons.Default.Person, "プロフィール")
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                val navBackStackEntry by navController.currentBackStackEntryAsState()
                items.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, item.label) },
                        label = { Text(item.label) },
                        selected = navBackStackEntry?.destination?.route == item.route::class.qualifiedName,
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
        AppNavHost(navController, Modifier.padding(padding))
    }
}
```

---

## まとめ

- 各featureモジュールで`NavGraphBuilder`拡張関数定義
- ルート定義は`core:navigation`に集約
- featureモジュールはNavControllerに依存しない
- appモジュールのNavHostで全featureを結合
- コールバック(onNavigateTo*)で画面間の依存を逆転
- BottomNavigationはappモジュールで統合

---

8種類のAndroidアプリテンプレート（マルチモジュール設計対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [マルチモジュールガイド](https://zenn.dev/myougatheaxo/articles/android-multi-module-2026)
- [型安全Navigationガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
- [Clean Architectureガイド](https://zenn.dev/myougatheaxo/articles/android-app-architecture-clean-2026)
