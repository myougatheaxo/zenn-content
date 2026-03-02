---
title: "NavigationDrawer完全ガイド — ModalDrawer/PermanentDrawer/DismissibleDrawer"
emoji: "📂"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**NavigationDrawer**（ModalNavigationDrawer、PermanentNavigationDrawer、DismissibleNavigationDrawer）を解説します。

---

## ModalNavigationDrawer

```kotlin
@Composable
fun ModalDrawerExample() {
    val drawerState = rememberDrawerState(initialValue = DrawerValue.Closed)
    val scope = rememberCoroutineScope()
    var selectedItem by remember { mutableIntStateOf(0) }

    val items = listOf(
        Triple("ホーム", Icons.Default.Home, Icons.Outlined.Home),
        Triple("設定", Icons.Default.Settings, Icons.Outlined.Settings),
        Triple("お気に入り", Icons.Default.Favorite, Icons.Outlined.FavoriteBorder),
        Triple("ヘルプ", Icons.Default.Info, Icons.Outlined.Info)
    )

    ModalNavigationDrawer(
        drawerState = drawerState,
        drawerContent = {
            ModalDrawerSheet {
                Spacer(Modifier.height(12.dp))
                Text("マイアプリ", Modifier.padding(16.dp), style = MaterialTheme.typography.titleLarge)
                HorizontalDivider(Modifier.padding(vertical = 8.dp))
                items.forEachIndexed { index, (label, filled, outlined) ->
                    NavigationDrawerItem(
                        icon = { Icon(if (selectedItem == index) filled else outlined, null) },
                        label = { Text(label) },
                        selected = selectedItem == index,
                        onClick = {
                            selectedItem = index
                            scope.launch { drawerState.close() }
                        },
                        modifier = Modifier.padding(NavigationDrawerItemDefaults.ItemPadding)
                    )
                }
            }
        }
    ) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text(items[selectedItem].first) },
                    navigationIcon = {
                        IconButton(onClick = { scope.launch { drawerState.open() } }) {
                            Icon(Icons.Default.Menu, "メニュー")
                        }
                    }
                )
            }
        ) { padding ->
            Box(Modifier.padding(padding).fillMaxSize(), contentAlignment = Alignment.Center) {
                Text("${items[selectedItem].first}のコンテンツ")
            }
        }
    }
}
```

---

## PermanentNavigationDrawer

```kotlin
@Composable
fun PermanentDrawerExample() {
    var selectedItem by remember { mutableIntStateOf(0) }
    val items = listOf("ホーム", "検索", "設定")

    PermanentNavigationDrawer(
        drawerContent = {
            PermanentDrawerSheet(Modifier.width(240.dp)) {
                Spacer(Modifier.height(12.dp))
                items.forEachIndexed { index, label ->
                    NavigationDrawerItem(
                        label = { Text(label) },
                        selected = selectedItem == index,
                        onClick = { selectedItem = index },
                        modifier = Modifier.padding(horizontal = 12.dp)
                    )
                }
            }
        }
    ) {
        Box(Modifier.fillMaxSize().padding(16.dp)) {
            Text("${items[selectedItem]}のコンテンツ")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ModalNavigationDrawer` | オーバーレイ型 |
| `PermanentNavigationDrawer` | 常時表示型 |
| `DismissibleNavigationDrawer` | 押し出し型 |
| `NavigationDrawerItem` | メニュー項目 |

- `ModalNavigationDrawer`でスマホ向けドロワー
- `PermanentNavigationDrawer`でタブレット/デスクトップ
- `DrawerState`でプログラム的に開閉
- `NavigationDrawerItem`でMaterial3準拠のメニュー

---

8種類のAndroidアプリテンプレート（ナビゲーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
- [Bottom Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
