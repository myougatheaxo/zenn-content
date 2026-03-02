---
title: "画面遷移アニメーションガイド — Navigation Compose Transition"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Navigation Composeの**画面遷移アニメーション**（enterTransition、exitTransition、共有要素）を解説します。

---

## 基本の遷移アニメーション

```kotlin
NavHost(
    navController = navController,
    startDestination = "home",
    enterTransition = { fadeIn(animationSpec = tween(300)) },
    exitTransition = { fadeOut(animationSpec = tween(300)) }
) {
    composable("home") { HomeScreen() }
    composable("detail") { DetailScreen() }
}
```

---

## スライドアニメーション

```kotlin
NavHost(
    navController = navController,
    startDestination = "home"
) {
    composable(
        "home",
        enterTransition = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            )
        },
        exitTransition = {
            slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            )
        },
        popEnterTransition = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Right,
                animationSpec = tween(300)
            )
        },
        popExitTransition = {
            slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Right,
                animationSpec = tween(300)
            )
        }
    ) {
        HomeScreen()
    }

    composable(
        "detail",
        enterTransition = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            )
        },
        popExitTransition = {
            slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Right,
                animationSpec = tween(300)
            )
        }
    ) {
        DetailScreen()
    }
}
```

---

## スケール + フェードの組み合わせ

```kotlin
composable(
    "detail",
    enterTransition = {
        scaleIn(
            initialScale = 0.9f,
            animationSpec = tween(300)
        ) + fadeIn(animationSpec = tween(300))
    },
    exitTransition = {
        scaleOut(
            targetScale = 1.1f,
            animationSpec = tween(300)
        ) + fadeOut(animationSpec = tween(300))
    }
) {
    DetailScreen()
}
```

---

## ルートごとの条件分岐

```kotlin
composable(
    "detail",
    enterTransition = {
        when (initialState.destination.route) {
            "home" -> slideIntoContainer(
                AnimatedContentTransitionScope.SlideDirection.Up,
                tween(400)
            )
            "search" -> fadeIn(tween(300))
            else -> fadeIn(tween(300))
        }
    }
) {
    DetailScreen()
}
```

---

## 共有遷移パターン

```kotlin
// 共通のアニメーション定義
object NavTransitions {
    val slideEnter: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition = {
        slideIntoContainer(
            AnimatedContentTransitionScope.SlideDirection.Left,
            tween(350, easing = FastOutSlowInEasing)
        )
    }

    val slideExit: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition = {
        slideOutOfContainer(
            AnimatedContentTransitionScope.SlideDirection.Left,
            tween(350, easing = FastOutSlowInEasing)
        )
    }

    val slidePopEnter: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition = {
        slideIntoContainer(
            AnimatedContentTransitionScope.SlideDirection.Right,
            tween(350, easing = FastOutSlowInEasing)
        )
    }

    val slidePopExit: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition = {
        slideOutOfContainer(
            AnimatedContentTransitionScope.SlideDirection.Right,
            tween(350, easing = FastOutSlowInEasing)
        )
    }

    val modalEnter: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition = {
        slideIntoContainer(
            AnimatedContentTransitionScope.SlideDirection.Up,
            tween(400)
        ) + fadeIn(tween(400))
    }

    val modalExit: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition = {
        slideOutOfContainer(
            AnimatedContentTransitionScope.SlideDirection.Down,
            tween(400)
        ) + fadeOut(tween(400))
    }
}

// 使用
composable(
    "detail",
    enterTransition = NavTransitions.slideEnter,
    exitTransition = NavTransitions.slideExit,
    popEnterTransition = NavTransitions.slidePopEnter,
    popExitTransition = NavTransitions.slidePopExit
) {
    DetailScreen()
}
```

---

## まとめ

- `enterTransition`/`exitTransition`で進む時のアニメーション
- `popEnterTransition`/`popExitTransition`で戻る時のアニメーション
- `slideIntoContainer`/`slideOutOfContainer`でスライド
- `fadeIn`/`fadeOut`/`scaleIn`/`scaleOut`を`+`で組み合わせ
- `initialState.destination.route`で遷移元による分岐
- 共通オブジェクトに定義して再利用

---

8種類のAndroidアプリテンプレート（画面遷移アニメーション設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [型安全Navigationガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
