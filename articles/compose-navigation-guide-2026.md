---
title: "Jetpack Compose Navigation入門 — 画面遷移を最速で理解する"
emoji: "🧭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

AIでAndroidアプリを作ると、画面が2つ以上あるアプリには必ず**Navigation**が使われます。「NavHost？NavController？」と混乱しがちですが、実は覚えることは3つだけです。

---

## 3つの登場人物

| 名前 | 役割 |
|------|------|
| **NavController** | 画面遷移を管理する司令塔 |
| **NavHost** | 画面を表示するコンテナ |
| **composable()** | 各画面の定義 |

---

## 最小構成

```kotlin
@Composable
fun MyApp() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToDetail = { id ->
                    navController.navigate("detail/$id")
                }
            )
        }
        composable("detail/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: ""
            DetailScreen(id = id)
        }
    }
}
```

### ポイント

1. `rememberNavController()`で1つだけ作る
2. `NavHost`に`startDestination`を指定
3. `composable("ルート名")`で画面を登録
4. `navController.navigate("ルート名")`で遷移

---

## 引数の渡し方

### パス引数（必須）

```kotlin
composable("detail/{habitId}") { backStackEntry ->
    val habitId = backStackEntry.arguments?.getString("habitId")
    DetailScreen(habitId = habitId ?: "")
}

// 遷移
navController.navigate("detail/abc123")
```

### クエリ引数（オプション）

```kotlin
composable(
    route = "list?sortBy={sortBy}",
    arguments = listOf(
        navArgument("sortBy") {
            defaultValue = "name"
            type = NavType.StringType
        }
    )
) { backStackEntry ->
    val sortBy = backStackEntry.arguments?.getString("sortBy") ?: "name"
    ListScreen(sortBy = sortBy)
}
```

---

## バックスタック管理

```kotlin
// 通常の遷移（バックスタックに積まれる）
navController.navigate("settings")

// バックスタックをクリアして遷移（ログイン→ホームなど）
navController.navigate("home") {
    popUpTo("login") { inclusive = true }
}

// 前の画面に戻る
navController.popBackStack()
```

### よくあるパターン

| シーン | コード |
|--------|--------|
| 通常遷移 | `navigate("route")` |
| 戻る | `popBackStack()` |
| ログイン後 | `navigate("home") { popUpTo("login") { inclusive = true } }` |
| タブ切り替え | `navigate("tab") { popUpTo("home") { saveState = true }; launchSingleTop = true; restoreState = true }` |

---

## BottomNavigationとの組み合わせ

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()

    Scaffold(
        bottomBar = {
            NavigationBar {
                val currentRoute = navController
                    .currentBackStackEntryAsState().value?.destination?.route

                NavigationBarItem(
                    icon = { Icon(Icons.Default.Home, "Home") },
                    label = { Text("ホーム") },
                    selected = currentRoute == "home",
                    onClick = {
                        navController.navigate("home") {
                            popUpTo("home") { saveState = true }
                            launchSingleTop = true
                            restoreState = true
                        }
                    }
                )
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Settings, "Settings") },
                    label = { Text("設定") },
                    selected = currentRoute == "settings",
                    onClick = {
                        navController.navigate("settings") {
                            popUpTo("home") { saveState = true }
                            launchSingleTop = true
                            restoreState = true
                        }
                    }
                )
            }
        }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(padding)
        ) {
            composable("home") { HomeScreen() }
            composable("settings") { SettingsScreen() }
        }
    }
}
```

---

## AIが生成するNavigationの特徴

| 項目 | Claude Codeの実装 |
|------|-----------------|
| ルート定義 | 文字列リテラル |
| 引数 | パス引数（/{id}） |
| BottomNav | NavigationBar + NavigationBarItem |
| バックスタック | popUpTo + launchSingleTop |
| Type Safety | 文字列ベース（Navigation 2.8+の型安全は未採用） |

---

## まとめ

- Navigationは**NavController + NavHost + composable**の3つだけ
- 引数はパス引数（`{id}`）が基本
- バックスタックは`popUpTo`で制御
- AIが生成するコードはそのまま動く

---

8種類のAndroidアプリテンプレート（Navigation実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [DataStore移行ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-migration-2026)
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
