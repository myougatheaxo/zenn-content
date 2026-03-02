---
title: "型安全Navigation完全ガイド — Kotlin Serialization/Safe Args/Compose"
emoji: "🧭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**型安全Navigation**（Kotlin Serialization統合、@Serializable route、型安全引数、ネスト対応）を解説します。

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
data class ProductDetail(
    val productId: Int,
    val category: String = "all"
)

@Serializable
data class SearchResult(
    val query: String,
    val page: Int = 1,
    val filters: List<String> = emptyList()
)
```

---

## NavHost構成

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(
                onUserClick = { userId -> navController.navigate(UserDetail(userId)) },
                onSettingsClick = { navController.navigate(Settings) },
                onProductClick = { id, cat -> navController.navigate(ProductDetail(id, cat)) }
            )
        }

        composable<Settings> {
            SettingsScreen(onBack = { navController.popBackStack() })
        }

        composable<UserDetail> { backStackEntry ->
            val userDetail = backStackEntry.toRoute<UserDetail>()
            UserDetailScreen(userId = userDetail.userId)
        }

        composable<ProductDetail> { backStackEntry ->
            val product = backStackEntry.toRoute<ProductDetail>()
            ProductDetailScreen(
                productId = product.productId,
                category = product.category
            )
        }
    }
}
```

---

## ネストNavigation

```kotlin
@Serializable object AuthGraph
@Serializable object Login
@Serializable object Register

@Serializable object MainGraph
@Serializable object Feed
@Serializable object Profile

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
                    onRegister = { navController.navigate(Register) }
                )
            }
            composable<Register> { RegisterScreen() }
        }

        navigation<MainGraph>(startDestination = Feed) {
            composable<Feed> { FeedScreen() }
            composable<Profile> { ProfileScreen() }
        }
    }
}
```

---

## DeepLink対応

```kotlin
composable<ProductDetail>(
    deepLinks = listOf(
        navDeepLink<ProductDetail>(basePath = "https://example.com/product")
    )
) { backStackEntry ->
    val product = backStackEntry.toRoute<ProductDetail>()
    ProductDetailScreen(product.productId, product.category)
}
// https://example.com/product/123?category=electronics → ProductDetail(123, "electronics")
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| ルート定義 | `@Serializable data class` |
| 遷移 | `navController.navigate(Route())` |
| 引数取得 | `backStackEntry.toRoute<T>()` |
| ネスト | `navigation<Graph>()` |
| DeepLink | `navDeepLink<T>()` |

- `@Serializable`でルートを型安全に定義
- 文字列ベースのルート定義が不要に
- コンパイル時に引数の型チェック
- Navigation 2.8+で利用可能

---

8種類のAndroidアプリテンプレート（型安全Navigation採用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation基本](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [Navigation遷移アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-animation-2026)
- [Deep Link](https://zenn.dev/myougatheaxo/articles/android-compose-app-links-deep-link-2026)
