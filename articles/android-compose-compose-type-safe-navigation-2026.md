---
title: "Compose Type-Safe Navigation完全ガイド — @Serializable/型安全ルート/引数渡し"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**Compose Type-Safe Navigation**（@Serializable route、型安全引数、ネスト、Kotlin Serialization連携）を解説します。

---

## 型安全ルート定義

```kotlin
// ルート定義（@Serializableで型安全に）
@Serializable object Home
@Serializable data class Detail(val id: String)
@Serializable data class Search(val query: String = "")
@Serializable object Settings

// NavHost
@Composable
fun AppNavHost() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(
                onItemClick = { id -> navController.navigate(Detail(id)) },
                onSearchClick = { navController.navigate(Search()) },
                onSettingsClick = { navController.navigate(Settings) }
            )
        }

        composable<Detail> { backStackEntry ->
            val detail: Detail = backStackEntry.toRoute()
            DetailScreen(detail.id)
        }

        composable<Search> { backStackEntry ->
            val search: Search = backStackEntry.toRoute()
            SearchScreen(search.query)
        }

        composable<Settings> {
            SettingsScreen()
        }
    }
}
```

---

## ネストナビゲーション

```kotlin
@Serializable object AuthGraph
@Serializable object Login
@Serializable object Register

@Serializable object MainGraph
@Serializable object Feed
@Serializable data class Profile(val userId: String)

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = AuthGraph) {
        navigation<AuthGraph>(startDestination = Login) {
            composable<Login> {
                LoginScreen(
                    onLoginSuccess = {
                        navController.navigate(MainGraph) {
                            popUpTo(AuthGraph) { inclusive = true }
                        }
                    },
                    onRegisterClick = { navController.navigate(Register) }
                )
            }
            composable<Register> { RegisterScreen() }
        }

        navigation<MainGraph>(startDestination = Feed) {
            composable<Feed> {
                FeedScreen(onProfileClick = { userId ->
                    navController.navigate(Profile(userId))
                })
            }
            composable<Profile> { backStackEntry ->
                val profile: Profile = backStackEntry.toRoute()
                ProfileScreen(profile.userId)
            }
        }
    }
}
```

---

## BottomNavigation連携

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val items = listOf(Home, Search(), Settings)
    val labels = listOf("ホーム", "検索", "設定")
    val icons = listOf(Icons.Default.Home, Icons.Default.Search, Icons.Default.Settings)

    Scaffold(
        bottomBar = {
            NavigationBar {
                val navBackStackEntry by navController.currentBackStackEntryAsState()
                items.forEachIndexed { index, route ->
                    NavigationBarItem(
                        selected = navBackStackEntry?.toRoute<Any>() == route,
                        onClick = {
                            navController.navigate(route) {
                                popUpTo(navController.graph.findStartDestination().id) { saveState = true }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        icon = { Icon(icons[index], labels[index]) },
                        label = { Text(labels[index]) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(navController, startDestination = Home, Modifier.padding(padding)) {
            composable<Home> { HomeScreen() }
            composable<Search> { SearchScreen() }
            composable<Settings> { SettingsScreen() }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@Serializable` | 型安全ルート定義 |
| `toRoute()` | 引数の型安全取得 |
| `composable<T>` | ルート登録 |
| `navigation<T>` | ネストグラフ |

- `@Serializable`でコンパイル時に型チェック
- `toRoute<T>()`でBackStackEntryから型安全に引数取得
- 文字列ルートと比べてタイポがなくリファクタリング安全
- ネストナビゲーションも型安全に定義可能

---

8種類のAndroidアプリテンプレート（Navigation対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-2026)
- [Compose DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-deep-link-2026)
- [Compose BottomNavigation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-navigation-2026)
