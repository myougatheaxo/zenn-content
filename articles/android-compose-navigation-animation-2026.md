---
title: "Navigation遷移アニメーション完全ガイド — enterTransition/SharedElement"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**Navigation遷移アニメーション**（enterTransition/exitTransition、SharedElement、カスタムアニメーション、条件分岐）を解説します。

---

## 基本の遷移アニメーション

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "home") {
        composable(
            route = "home",
            enterTransition = { fadeIn(animationSpec = tween(300)) },
            exitTransition = { fadeOut(animationSpec = tween(300)) }
        ) {
            HomeScreen(onNavigate = { navController.navigate("detail/$it") })
        }

        composable(
            route = "detail/{id}",
            enterTransition = {
                slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Left, tween(300))
            },
            exitTransition = {
                slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Left, tween(300))
            },
            popEnterTransition = {
                slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Right, tween(300))
            },
            popExitTransition = {
                slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Right, tween(300))
            }
        ) { backStackEntry ->
            DetailScreen(backStackEntry.arguments?.getString("id") ?: "")
        }
    }
}
```

---

## SharedElement遷移

```kotlin
@Composable
fun SharedElementExample() {
    val navController = rememberNavController()

    SharedTransitionLayout {
        NavHost(navController, startDestination = "list") {
            composable("list") {
                LazyColumn {
                    items(sampleItems) { item ->
                        Row(
                            Modifier.clickable { navController.navigate("detail/${item.id}") }
                                .padding(16.dp)
                        ) {
                            AsyncImage(
                                model = item.imageUrl,
                                contentDescription = null,
                                modifier = Modifier
                                    .size(60.dp)
                                    .sharedElement(
                                        rememberSharedContentState(key = "image-${item.id}"),
                                        animatedVisibilityScope = this@composable
                                    )
                            )
                            Text(
                                item.title,
                                modifier = Modifier
                                    .padding(start = 16.dp)
                                    .sharedElement(
                                        rememberSharedContentState(key = "title-${item.id}"),
                                        animatedVisibilityScope = this@composable
                                    )
                            )
                        }
                    }
                }
            }

            composable("detail/{id}") { backStackEntry ->
                val id = backStackEntry.arguments?.getString("id") ?: return@composable
                val item = sampleItems.first { it.id == id }

                Column(Modifier.fillMaxSize()) {
                    AsyncImage(
                        model = item.imageUrl,
                        contentDescription = null,
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(300.dp)
                            .sharedElement(
                                rememberSharedContentState(key = "image-${item.id}"),
                                animatedVisibilityScope = this@composable
                            )
                    )
                    Text(
                        item.title,
                        style = MaterialTheme.typography.headlineMedium,
                        modifier = Modifier
                            .padding(16.dp)
                            .sharedElement(
                                rememberSharedContentState(key = "title-${item.id}"),
                                animatedVisibilityScope = this@composable
                            )
                    )
                }
            }
        }
    }
}
```

---

## カスタムアニメーション

```kotlin
composable(
    route = "settings",
    enterTransition = {
        scaleIn(initialScale = 0.8f, animationSpec = tween(300)) +
            fadeIn(animationSpec = tween(300))
    },
    exitTransition = {
        scaleOut(targetScale = 0.8f, animationSpec = tween(300)) +
            fadeOut(animationSpec = tween(300))
    }
) {
    SettingsScreen()
}
```

---

## まとめ

| アニメーション | API |
|--------------|-----|
| スライド | `slideIntoContainer` |
| フェード | `fadeIn` / `fadeOut` |
| スケール | `scaleIn` / `scaleOut` |
| 共有要素 | `sharedElement()` |
| 組み合わせ | `+` 演算子 |

- `enterTransition`/`exitTransition`で遷移アニメーション
- `popEnterTransition`で戻り遷移を別制御
- `SharedTransitionLayout`で共有要素アニメーション
- `tween`/`spring`でイージング調整

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation基本](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [SharedElement遷移](https://zenn.dev/myougatheaxo/articles/android-compose-shared-element-2026)
