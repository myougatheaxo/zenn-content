---
title: "Compose Navigation引数パターン集 — 型安全なデータ渡し"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

Compose Navigationで画面間にデータを渡すパターンを**全網羅**します。String、Int、Optional、Parcelable、複雑なオブジェクトまで。

---

## 基本: String引数

```kotlin
NavHost(navController, startDestination = "home") {
    composable("home") {
        HomeScreen(
            onItemClick = { id ->
                navController.navigate("detail/$id")
            }
        )
    }

    composable(
        route = "detail/{itemId}",
        arguments = listOf(
            navArgument("itemId") { type = NavType.StringType }
        )
    ) { backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
        DetailScreen(itemId)
    }
}
```

---

## Int引数

```kotlin
composable(
    route = "profile/{userId}",
    arguments = listOf(
        navArgument("userId") { type = NavType.IntType }
    )
) { backStackEntry ->
    val userId = backStackEntry.arguments?.getInt("userId") ?: 0
    ProfileScreen(userId)
}

// 遷移
navController.navigate("profile/42")
```

---

## Optional引数（クエリパラメータ）

```kotlin
composable(
    route = "search?query={query}&category={category}",
    arguments = listOf(
        navArgument("query") {
            type = NavType.StringType
            defaultValue = ""
        },
        navArgument("category") {
            type = NavType.StringType
            defaultValue = "all"
        }
    )
) { backStackEntry ->
    val query = backStackEntry.arguments?.getString("query") ?: ""
    val category = backStackEntry.arguments?.getString("category") ?: "all"
    SearchScreen(query, category)
}

// 遷移パターン
navController.navigate("search")                          // デフォルト値
navController.navigate("search?query=kotlin")             // queryのみ
navController.navigate("search?query=kotlin&category=lib") // 両方指定
```

---

## Boolean引数

```kotlin
composable(
    route = "editor/{isNew}",
    arguments = listOf(
        navArgument("isNew") { type = NavType.BoolType }
    )
) { backStackEntry ->
    val isNew = backStackEntry.arguments?.getBoolean("isNew") ?: true
    EditorScreen(isNew)
}

navController.navigate("editor/true")   // 新規
navController.navigate("editor/false")  // 編集
```

---

## 複数の必須引数

```kotlin
composable(
    route = "task/{taskId}/{categoryId}",
    arguments = listOf(
        navArgument("taskId") { type = NavType.IntType },
        navArgument("categoryId") { type = NavType.IntType }
    )
) { backStackEntry ->
    val taskId = backStackEntry.arguments?.getInt("taskId") ?: 0
    val categoryId = backStackEntry.arguments?.getInt("categoryId") ?: 0
    TaskDetailScreen(taskId, categoryId)
}
```

---

## 結果の受け取り（戻り値）

```kotlin
// 画面Aで結果を受け取る
val result = navController.currentBackStackEntry
    ?.savedStateHandle
    ?.getStateFlow<String>("result", "")
    ?.collectAsState()

// 画面Bで結果をセット
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("result", "selected_item_id")
navController.popBackStack()
```

---

## ルート定義の整理

```kotlin
object Routes {
    const val HOME = "home"
    const val SETTINGS = "settings"

    fun detail(id: String) = "detail/$id"
    fun profile(userId: Int) = "profile/$userId"
    fun search(query: String = "", category: String = "all") =
        "search?query=$query&category=$category"
}

// 使い方
navController.navigate(Routes.detail("abc123"))
navController.navigate(Routes.profile(42))
navController.navigate(Routes.search(query = "kotlin"))
```

---

## まとめ

| パターン | ルート | 用途 |
|---------|--------|------|
| 必須String | `route/{id}` | 詳細画面 |
| 必須Int | `route/{num}` | ID指定 |
| Optional | `route?key={val}` | 検索・フィルタ |
| 複数引数 | `route/{a}/{b}` | 複合キー |
| 戻り値 | `savedStateHandle` | 選択結果 |

---

8種類のAndroidアプリテンプレート（Navigation設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [ディープリンク完全ガイド](https://zenn.dev/myougatheaxo/articles/android-deeplink-guide-2026)
- [Kotlin sealed class完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
