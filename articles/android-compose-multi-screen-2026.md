---
title: "マルチスクリーン対応ガイド — Composeでタブレット/折りたたみ対応"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

Composeで**タブレット・折りたたみデバイス**に対応したレスポンシブUIを実装する方法を解説します。

---

## 画面サイズの判定

```kotlin
enum class WindowSizeClass { COMPACT, MEDIUM, EXPANDED }

@Composable
fun rememberWindowSizeClass(): WindowSizeClass {
    val configuration = LocalConfiguration.current
    return when {
        configuration.screenWidthDp < 600 -> WindowSizeClass.COMPACT
        configuration.screenWidthDp < 840 -> WindowSizeClass.MEDIUM
        else -> WindowSizeClass.EXPANDED
    }
}
```

---

## レスポンシブレイアウト

```kotlin
@Composable
fun AdaptiveLayout() {
    val windowSize = rememberWindowSizeClass()

    when (windowSize) {
        WindowSizeClass.COMPACT -> PhoneLayout()
        WindowSizeClass.MEDIUM -> TabletLayout()
        WindowSizeClass.EXPANDED -> DesktopLayout()
    }
}

@Composable
fun PhoneLayout() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = "list") {
        composable("list") { ItemList(onItemClick = { navController.navigate("detail/$it") }) }
        composable("detail/{id}") { DetailScreen(it.arguments?.getString("id")) }
    }
}

@Composable
fun TabletLayout() {
    var selectedId by remember { mutableStateOf<String?>(null) }
    Row(Modifier.fillMaxSize()) {
        ItemList(onItemClick = { selectedId = it }, modifier = Modifier.weight(1f))
        selectedId?.let {
            DetailScreen(it, modifier = Modifier.weight(2f))
        } ?: Box(Modifier.weight(2f).fillMaxHeight(), contentAlignment = Alignment.Center) {
            Text("項目を選択してください")
        }
    }
}
```

---

## レスポンシブNavigation

```kotlin
@Composable
fun AdaptiveNavigation(selectedIndex: Int, onItemSelected: (Int) -> Unit, content: @Composable () -> Unit) {
    val windowSize = rememberWindowSizeClass()
    val items = listOf(
        NavigationItem("ホーム", Icons.Default.Home),
        NavigationItem("検索", Icons.Default.Search),
        NavigationItem("設定", Icons.Default.Settings)
    )

    when (windowSize) {
        WindowSizeClass.COMPACT -> {
            Scaffold(bottomBar = {
                NavigationBar {
                    items.forEachIndexed { index, item ->
                        NavigationBarItem(
                            icon = { Icon(item.icon, item.label) },
                            label = { Text(item.label) },
                            selected = selectedIndex == index,
                            onClick = { onItemSelected(index) }
                        )
                    }
                }
            }) { padding -> Box(Modifier.padding(padding)) { content() } }
        }
        WindowSizeClass.MEDIUM -> {
            Row {
                NavigationRail {
                    items.forEachIndexed { index, item ->
                        NavigationRailItem(
                            icon = { Icon(item.icon, item.label) },
                            label = { Text(item.label) },
                            selected = selectedIndex == index,
                            onClick = { onItemSelected(index) }
                        )
                    }
                }
                content()
            }
        }
        WindowSizeClass.EXPANDED -> {
            PermanentNavigationDrawer(drawerContent = {
                PermanentDrawerSheet(Modifier.width(240.dp)) {
                    items.forEachIndexed { index, item ->
                        NavigationDrawerItem(
                            icon = { Icon(item.icon, item.label) },
                            label = { Text(item.label) },
                            selected = selectedIndex == index,
                            onClick = { onItemSelected(index) }
                        )
                    }
                }
            }) { content() }
        }
    }
}

data class NavigationItem(val label: String, val icon: ImageVector)
```

---

## まとめ

- Compact(<600dp): BottomNavigation + 1カラム
- Medium(<840dp): NavigationRail + 2カラム
- Expanded(840dp+): PermanentDrawer + マルチカラム
- `GridCells.Fixed(n)`でグリッド列数を動的変更

---

8種類のAndroidアプリテンプレート（レスポンシブ設計対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-basic-2026)
- [NavigationDrawerガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-drawer-2026)
- [LazyVerticalGridガイド](https://zenn.dev/myougatheaxo/articles/compose-lazy-grid-2026)
