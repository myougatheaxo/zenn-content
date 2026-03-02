---
title: "Navigation Animation完全ガイド — enterTransition/exitTransition/画面遷移アニメーション"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**Navigation Animation**（enterTransition、exitTransition、画面遷移アニメーション）を解説します。

---

## 画面遷移アニメーション

```kotlin
@Composable
fun AnimatedNavHost() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home",
        enterTransition = { fadeIn(tween(300)) + slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Start, tween(300)) },
        exitTransition = { fadeOut(tween(300)) + slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Start, tween(300)) },
        popEnterTransition = { fadeIn(tween(300)) + slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.End, tween(300)) },
        popExitTransition = { fadeOut(tween(300)) + slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.End, tween(300)) }
    ) {
        composable("home") { HomeScreen(navController) }
        composable("detail/{id}") { DetailScreen(navController) }
    }
}
```

---

## 個別画面のアニメーション

```kotlin
composable(
    route = "settings",
    enterTransition = {
        slideIntoContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Up,
            animationSpec = tween(500)
        )
    },
    exitTransition = {
        slideOutOfContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Down,
            animationSpec = tween(500)
        )
    }
) {
    SettingsScreen()
}

// フェードのみ
composable(
    route = "profile",
    enterTransition = { fadeIn(tween(500)) },
    exitTransition = { fadeOut(tween(500)) }
) {
    ProfileScreen()
}
```

---

## 遷移元に応じた分岐

```kotlin
composable(
    route = "detail/{id}",
    enterTransition = {
        when (initialState.destination.route) {
            "home" -> slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Start)
            "search" -> fadeIn() + scaleIn(initialScale = 0.9f)
            else -> fadeIn()
        }
    }
) {
    DetailScreen()
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `enterTransition` | 表示アニメーション |
| `exitTransition` | 退場アニメーション |
| `popEnterTransition` | 戻り表示 |
| `popExitTransition` | 戻り退場 |

- NavHost全体のデフォルトアニメーション設定
- 個別composableでオーバーライド可能
- `slideIntoContainer`で方向指定スライド
- `initialState`で遷移元に応じた分岐

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-2026)
- [SharedElementTransition](https://zenn.dev/myougatheaxo/articles/android-compose-shared-element-transition-2026)
- [AnimatedContent](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-content-2026)
