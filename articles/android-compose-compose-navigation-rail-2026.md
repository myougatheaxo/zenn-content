---
title: "NavigationRail完全ガイド — タブレット対応/FAB配置/レスポンシブナビ"
emoji: "🚂"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**NavigationRail**（タブレット向けサイドナビ、FAB配置、WindowSizeClass連携）を解説します。

---

## NavigationRail基本

```kotlin
@Composable
fun NavigationRailScreen() {
    var selectedIndex by remember { mutableIntStateOf(0) }
    val items = listOf(
        "ホーム" to Icons.Default.Home,
        "検索" to Icons.Default.Search,
        "設定" to Icons.Default.Settings
    )

    Row(Modifier.fillMaxSize()) {
        NavigationRail(
            header = {
                FloatingActionButton(onClick = {}) {
                    Icon(Icons.Default.Add, "追加")
                }
            }
        ) {
            Spacer(Modifier.weight(1f))
            items.forEachIndexed { index, (label, icon) ->
                NavigationRailItem(
                    selected = selectedIndex == index,
                    onClick = { selectedIndex = index },
                    icon = { Icon(icon, label) },
                    label = { Text(label) }
                )
            }
            Spacer(Modifier.weight(1f))
        }

        Box(Modifier.weight(1f).fillMaxHeight().padding(16.dp)) {
            Text("${items[selectedIndex].first}画面", style = MaterialTheme.typography.headlineMedium)
        }
    }
}
```

---

## レスポンシブナビ

```kotlin
@Composable
fun ResponsiveNavigation() {
    val windowSizeClass = currentWindowAdaptiveInfo().windowSizeClass

    when {
        windowSizeClass.windowWidthSizeClass == WindowWidthSizeClass.EXPANDED -> {
            NavigationRailLayout()
        }
        else -> {
            BottomNavigationLayout()
        }
    }
}

@Composable
fun NavigationRailLayout() {
    var selected by remember { mutableIntStateOf(0) }
    Row(Modifier.fillMaxSize()) {
        NavigationRail {
            listOf("ホーム" to Icons.Default.Home, "検索" to Icons.Default.Search, "設定" to Icons.Default.Settings)
                .forEachIndexed { i, (label, icon) ->
                    NavigationRailItem(selected == i, { selected = i }, { Icon(icon, label) }, label = { Text(label) })
                }
        }
        VerticalDivider()
        Box(Modifier.weight(1f).padding(16.dp)) { Text("コンテンツ") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `NavigationRail` | サイドナビゲーション |
| `NavigationRailItem` | ナビアイテム |
| `header` | FAB等のヘッダー |
| `WindowSizeClass` | 画面サイズ判定 |

- `NavigationRail`でタブレット向けサイドナビ
- `header`にFABを配置可能
- `WindowSizeClass`でBottomNavとの切替
- Material3のレスポンシブデザインガイドに準拠

---

8種類のAndroidアプリテンプレート（レスポンシブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-navigation-2026)
- [NavigationDrawer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-drawer-2026)
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
