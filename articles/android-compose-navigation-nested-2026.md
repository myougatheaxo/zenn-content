---
title: "Compose Navigation ネスト構成ガイド — nested graph/BottomNav連携"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

Compose Navigationの**ネスト構成**（nested graph、BottomNav連携、認証フロー、Deep Link）を解説します。

---

## 基本のNavHost

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(
                onNavigateToDetail = { id ->
                    navController.navigate("detail/$id")
                }
            )
        }
        composable(
            "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType })
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
            DetailScreen(itemId = itemId)
        }
    }
}
```

---

## Nested Graph（認証フロー）

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "auth") {
        // 認証グラフ
        navigation(startDestination = "login", route = "auth") {
            composable("login") {
                LoginScreen(
                    onLoginSuccess = {
                        navController.navigate("main") {
                            popUpTo("auth") { inclusive = true }
                        }
                    },
                    onNavigateToRegister = {
                        navController.navigate("register")
                    }
                )
            }
            composable("register") {
                RegisterScreen(
                    onRegisterSuccess = {
                        navController.navigate("main") {
                            popUpTo("auth") { inclusive = true }
                        }
                    }
                )
            }
        }

        // メイングラフ
        navigation(startDestination = "home", route = "main") {
            composable("home") { HomeScreen() }
            composable("profile") { ProfileScreen() }
            composable("settings") { SettingsScreen() }
        }
    }
}
```

---

## BottomNav + NavHost

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()

    val bottomItems = listOf(
        BottomNavItem("home", "ホーム", Icons.Default.Home),
        BottomNavItem("search", "検索", Icons.Default.Search),
        BottomNavItem("profile", "プロフィール", Icons.Default.Person)
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                val navBackStackEntry by navController.currentBackStackEntryAsState()
                val currentRoute = navBackStackEntry?.destination?.route

                bottomItems.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) },
                        selected = currentRoute == item.route,
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
            startDestination = "home",
            modifier = Modifier.padding(padding)
        ) {
            composable("home") { HomeScreen(navController) }
            composable("search") { SearchScreen(navController) }
            composable("profile") { ProfileScreen(navController) }
            composable("detail/{id}") { DetailScreen() }
        }
    }
}

data class BottomNavItem(val route: String, val label: String, val icon: ImageVector)
```

---

## 画面間データ受け渡し

```kotlin
// 前の画面に結果を返す
@Composable
fun EditScreen(navController: NavController) {
    var text by remember { mutableStateOf("") }

    Button(onClick = {
        navController.previousBackStackEntry
            ?.savedStateHandle
            ?.set("result", text)
        navController.popBackStack()
    }) {
        Text("保存して戻る")
    }
}

// 結果を受け取る
@Composable
fun ListScreen(navController: NavController) {
    val result = navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<String?>("result", null)
        ?.collectAsStateWithLifecycle()

    LaunchedEffect(result?.value) {
        result?.value?.let { data ->
            // 結果を処理
        }
    }
}
```

---

## Type-safe Navigation (Kotlin Serialization)

```kotlin
@Serializable
data class DetailRoute(val itemId: String)

@Serializable
object HomeRoute

@Serializable
object ProfileRoute

@Composable
fun TypeSafeNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            HomeScreen(
                onItemClick = { id ->
                    navController.navigate(DetailRoute(itemId = id))
                }
            )
        }
        composable<DetailRoute> { backStackEntry ->
            val route: DetailRoute = backStackEntry.toRoute()
            DetailScreen(itemId = route.itemId)
        }
    }
}
```

---

## まとめ

- `navigation()`でNested Graphを作成
- `popUpTo() { inclusive = true }`で認証画面をスタックから除去
- `saveState`/`restoreState`でBottomNavのタブ状態保持
- `savedStateHandle`で画面間のデータ受け渡し
- `launchSingleTop = true`で重複画面防止
- Kotlin Serializationで型安全なナビゲーション

---

8種類のAndroidアプリテンプレート（Navigation設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation引数](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-args-2026)
- [型安全Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
- [BottomNav+Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-scaffold-2026)
