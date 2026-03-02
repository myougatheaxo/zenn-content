---
title: "型安全Navigation完全ガイド — @Serializable/typeSafeNavigation/引数の型安全"
emoji: "🧭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**型安全Navigation**（@Serializable route、型安全な引数、ネストグラフ）を解説します。

---

## @Serializable Route定義

```kotlin
@Serializable
data object Home

@Serializable
data object Settings

@Serializable
data class Detail(val itemId: Long)

@Serializable
data class Search(val query: String = "")
```

---

## NavHost

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(
                onItemClick = { id -> navController.navigate(Detail(itemId = id)) },
                onSearchClick = { navController.navigate(Search()) }
            )
        }

        composable<Detail> { backStackEntry ->
            val detail: Detail = backStackEntry.toRoute()
            DetailScreen(
                itemId = detail.itemId,
                onBack = { navController.popBackStack() }
            )
        }

        composable<Search> { backStackEntry ->
            val search: Search = backStackEntry.toRoute()
            SearchScreen(initialQuery = search.query)
        }

        composable<Settings> {
            SettingsScreen()
        }
    }
}
```

---

## ネストグラフ

```kotlin
@Serializable
data object AuthGraph

@Serializable
data object Login

@Serializable
data object Register

@Composable
fun AppWithAuth() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = AuthGraph) {
        navigation<AuthGraph>(startDestination = Login) {
            composable<Login> {
                LoginScreen(
                    onLoginSuccess = {
                        navController.navigate(Home) {
                            popUpTo(AuthGraph) { inclusive = true }
                        }
                    },
                    onRegister = { navController.navigate(Register) }
                )
            }
            composable<Register> {
                RegisterScreen(onBack = { navController.popBackStack() })
            }
        }

        composable<Home> { HomeScreen() }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@Serializable` | 型安全route定義 |
| `composable<T>` | 型安全destination |
| `toRoute()` | 引数の型安全取得 |
| `navigation<T>` | 型安全ネストグラフ |

- `@Serializable`でコンパイル時型チェック
- 文字列routeが不要でタイポ防止
- `toRoute()`で引数を型安全に取得
- Navigation Compose 2.8+で使用可能

---

8種類のAndroidアプリテンプレート（Navigation実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-2026)
- [DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-deep-link-2026)
- [Navigation Result](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-result-2026)
