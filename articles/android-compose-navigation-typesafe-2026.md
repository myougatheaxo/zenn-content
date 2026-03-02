---
title: "型安全Navigationガイド — Kotlin Serializationで型安全な画面遷移"
emoji: "🧭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

Compose Navigationの**型安全なルート定義**（Kotlin Serialization連携）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.serialization")
}

dependencies {
    implementation("androidx.navigation:navigation-compose:2.8.5")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
}
```

---

## ルート定義

```kotlin
@Serializable
object Home

@Serializable
object Settings

@Serializable
data class UserDetail(val userId: String)

@Serializable
data class Search(val query: String = "")
```

---

## NavHostの構築

```kotlin
@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController = navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(
                onNavigateToUser = { userId ->
                    navController.navigate(UserDetail(userId))
                },
                onNavigateToSettings = {
                    navController.navigate(Settings)
                }
            )
        }

        composable<Settings> {
            SettingsScreen(onBack = { navController.popBackStack() })
        }

        composable<UserDetail> { backStackEntry ->
            val userDetail: UserDetail = backStackEntry.toRoute()
            UserDetailScreen(
                userId = userDetail.userId,
                onBack = { navController.popBackStack() }
            )
        }

        composable<Search> { backStackEntry ->
            val search: Search = backStackEntry.toRoute()
            SearchScreen(initialQuery = search.query)
        }
    }
}
```

---

## ネストされたナビゲーション

```kotlin
@Serializable
object MainGraph

@Serializable
object AuthGraph

@Serializable
object Login

@Serializable
object Register

@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController = navController, startDestination = AuthGraph) {
        navigation<AuthGraph>(startDestination = Login) {
            composable<Login> {
                LoginScreen(
                    onLoginSuccess = {
                        navController.navigate(MainGraph) {
                            popUpTo(AuthGraph) { inclusive = true }
                        }
                    },
                    onNavigateToRegister = {
                        navController.navigate(Register)
                    }
                )
            }
            composable<Register> {
                RegisterScreen(onBack = { navController.popBackStack() })
            }
        }

        navigation<MainGraph>(startDestination = Home) {
            composable<Home> {
                HomeScreen()
            }
            composable<Settings> {
                SettingsScreen()
            }
        }
    }
}
```

---

## 複雑な引数の受け渡し

```kotlin
@Serializable
data class ProductDetail(
    val productId: Long,
    val category: String,
    val fromSearch: Boolean = false
)

@Serializable
data class OrderConfirm(
    val productId: Long,
    val quantity: Int,
    val couponCode: String? = null
)

// 使用例
navController.navigate(
    ProductDetail(
        productId = 123L,
        category = "electronics",
        fromSearch = true
    )
)

navController.navigate(
    OrderConfirm(
        productId = 123L,
        quantity = 2,
        couponCode = "SAVE10"
    )
)
```

---

## BottomNavigation連携

```kotlin
@Serializable
object HomeTab

@Serializable
object SearchTab

@Serializable
object ProfileTab

data class BottomNavItem(
    val route: Any,
    val icon: ImageVector,
    val label: String
)

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val items = listOf(
        BottomNavItem(HomeTab, Icons.Default.Home, "ホーム"),
        BottomNavItem(SearchTab, Icons.Default.Search, "検索"),
        BottomNavItem(ProfileTab, Icons.Default.Person, "プロフィール")
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                val navBackStackEntry by navController.currentBackStackEntryAsState()
                items.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = item.label) },
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
        NavHost(
            navController = navController,
            startDestination = HomeTab,
            modifier = Modifier.padding(padding)
        ) {
            composable<HomeTab> { HomeContent() }
            composable<SearchTab> { SearchContent() }
            composable<ProfileTab> { ProfileContent() }
        }
    }
}
```

---

## まとめ

- `@Serializable`オブジェクト/データクラスでルート定義
- `composable<Route>`で型安全な画面登録
- `backStackEntry.toRoute<Route>()`で引数取得
- `navigation<Graph>`でネストされたナビゲーション
- デフォルト引数・nullable引数に対応
- BottomNavigationとの連携も型安全

---

8種類のAndroidアプリテンプレート（型安全Navigation設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [Navigation引数ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-args-2026)
- [BottomNavigation実装](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-2026)
