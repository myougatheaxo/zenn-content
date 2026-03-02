---
title: "レスポンシブレイアウト完全ガイド — タブレット/折りたたみ対応"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

**レスポンシブレイアウト**（WindowSizeClass、適応型レイアウト、タブレット対応、折りたたみデバイス）を解説します。

---

## WindowSizeClass

```kotlin
dependencies {
    implementation("androidx.compose.material3:material3-window-size-class")
}
```

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val windowSizeClass = calculateWindowSizeClass(this)
            AppContent(windowSizeClass)
        }
    }
}

@Composable
fun AppContent(windowSizeClass: WindowSizeClass) {
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> PhoneLayout()
        WindowWidthSizeClass.Medium -> TabletLayout()
        WindowWidthSizeClass.Expanded -> DesktopLayout()
    }
}
```

---

## List-Detail パターン

```kotlin
@Composable
fun ListDetailLayout(
    windowSizeClass: WindowSizeClass,
    items: List<Item>,
    selectedItem: Item?,
    onItemClick: (Item) -> Unit
) {
    if (windowSizeClass.widthSizeClass == WindowWidthSizeClass.Expanded) {
        Row(Modifier.fillMaxSize()) {
            ItemList(items, onItemClick, Modifier.weight(1f))
            VerticalDivider()
            selectedItem?.let {
                ItemDetail(it, Modifier.weight(2f))
            } ?: Box(Modifier.weight(2f), contentAlignment = Alignment.Center) {
                Text("アイテムを選択してください")
            }
        }
    } else {
        val navController = rememberNavController()
        NavHost(navController, "list") {
            composable("list") {
                ItemList(items, { onItemClick(it); navController.navigate("detail/${it.id}") })
            }
            composable("detail/{id}") { selectedItem?.let { ItemDetail(it) } }
        }
    }
}
```

---

## 適応型Grid

```kotlin
@Composable
fun AdaptiveGrid(items: List<Item>) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 160.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items, key = { it.id }) { item -> ItemCard(item) }
    }
}
```

---

## Navigation適応

```kotlin
@Composable
fun AdaptiveNavigation(windowSizeClass: WindowSizeClass, currentRoute: String, onNavigate: (String) -> Unit, content: @Composable (PaddingValues) -> Unit) {
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            Scaffold(bottomBar = {
                NavigationBar {
                    navItems.forEach { item ->
                        NavigationBarItem(selected = currentRoute == item.route, onClick = { onNavigate(item.route) }, icon = { Icon(item.icon, null) }, label = { Text(item.label) })
                    }
                }
            }, content = content)
        }
        else -> {
            Row(Modifier.fillMaxSize()) {
                NavigationRail {
                    navItems.forEach { item ->
                        NavigationRailItem(selected = currentRoute == item.route, onClick = { onNavigate(item.route) }, icon = { Icon(item.icon, null) }, label = { Text(item.label) })
                    }
                }
                content(PaddingValues())
            }
        }
    }
}
```

---

## まとめ

| 画面幅 | クラス | レイアウト |
|--------|--------|----------|
| < 600dp | Compact | BottomNav + 単一画面 |
| 600-840dp | Medium | NavigationRail + Grid |
| > 840dp | Expanded | NavigationRail + List-Detail |

- `WindowSizeClass`で画面サイズ判定
- `GridCells.Adaptive`で自動列数調整
- `NavigationRail`でタブレットナビゲーション

---

8種類のAndroidアプリテンプレート（レスポンシブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [ConstraintLayout](https://zenn.dev/myougatheaxo/articles/android-compose-constraint-layout-compose-2026)
