---
title: "Adaptive Layout完全ガイド — ListDetailPaneScaffold/NavigationSuiteScaffold"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "adaptive"]
published: true
---

## この記事で学べること

**Adaptive Layout**（ListDetailPaneScaffold、NavigationSuiteScaffold、適応型ナビゲーション）を解説します。

---

## ListDetailPaneScaffold

```kotlin
@OptIn(ExperimentalMaterial3AdaptiveApi::class)
@Composable
fun AdaptiveListDetail() {
    val navigator = rememberListDetailPaneScaffoldNavigator<String>()

    ListDetailPaneScaffold(
        directive = navigator.scaffoldDirective,
        value = navigator.scaffoldValue,
        listPane = {
            LazyColumn {
                items(20) { index ->
                    ListItem(
                        headlineContent = { Text("Item $index") },
                        modifier = Modifier.clickable {
                            navigator.navigateTo(ListDetailPaneScaffoldRole.Detail, "Item $index")
                        }
                    )
                }
            }
        },
        detailPane = {
            navigator.currentDestination?.content?.let { content ->
                Column(Modifier.padding(24.dp)) {
                    Text(content, style = MaterialTheme.typography.headlineMedium)
                    Spacer(Modifier.height(16.dp))
                    Text("$content の詳細コンテンツ")
                }
            } ?: Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                Text("アイテムを選択してください")
            }
        }
    )
}
```

---

## NavigationSuiteScaffold

```kotlin
@OptIn(ExperimentalMaterial3AdaptiveNavigationSuiteApi::class)
@Composable
fun AdaptiveNavigation() {
    var selectedItem by remember { mutableIntStateOf(0) }
    val items = listOf(
        Triple("ホーム", Icons.Default.Home, Icons.Outlined.Home),
        Triple("検索", Icons.Default.Search, Icons.Outlined.Search),
        Triple("設定", Icons.Default.Settings, Icons.Outlined.Settings)
    )

    NavigationSuiteScaffold(
        navigationSuiteItems = {
            items.forEachIndexed { index, (label, filledIcon, outlinedIcon) ->
                item(
                    selected = selectedItem == index,
                    onClick = { selectedItem = index },
                    icon = {
                        Icon(
                            if (selectedItem == index) filledIcon else outlinedIcon,
                            contentDescription = label
                        )
                    },
                    label = { Text(label) }
                )
            }
        }
    ) {
        when (selectedItem) {
            0 -> HomeScreen()
            1 -> SearchScreen()
            2 -> SettingsScreen()
        }
    }
}
```

---

## WindowSizeClass連携

```kotlin
@Composable
fun AdaptiveApp() {
    val windowSizeClass = calculateWindowSizeClass(LocalContext.current as Activity)

    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // スマホ: BottomNavigation
            CompactLayout()
        }
        WindowWidthSizeClass.Medium -> {
            // タブレット縦: NavigationRail
            MediumLayout()
        }
        WindowWidthSizeClass.Expanded -> {
            // タブレット横/デスクトップ: NavigationDrawer
            ExpandedLayout()
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ListDetailPaneScaffold` | リスト/詳細の2ペイン |
| `NavigationSuiteScaffold` | 適応型ナビゲーション |
| `WindowSizeClass` | 画面サイズ判定 |
| `scaffoldDirective` | ペイン配置制御 |

- `ListDetailPaneScaffold`でスマホ→2ペイン自動切替
- `NavigationSuiteScaffold`でBar/Rail/Drawer自動切替
- `WindowSizeClass`でCompact/Medium/Expanded分岐
- Material3 Adaptiveライブラリで統一的な適応UI

---

8種類のAndroidアプリテンプレート（レスポンシブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WindowSize](https://zenn.dev/myougatheaxo/articles/android-compose-compose-window-size-2026)
- [横画面/タブレット](https://zenn.dev/myougatheaxo/articles/android-compose-landscape-tablet-2026)
- [折りたたみデバイス](https://zenn.dev/myougatheaxo/articles/android-compose-foldable-device-2026)
