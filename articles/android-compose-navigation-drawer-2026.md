---
title: "NavigationDrawer完全ガイド — Composeでサイドメニューを実装"
emoji: "☰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**ModalNavigationDrawer**と**PermanentNavigationDrawer**の実装方法を解説します。

---

## ModalNavigationDrawer

```kotlin
@Composable
fun DrawerExample() {
    val drawerState = rememberDrawerState(DrawerValue.Closed)
    val scope = rememberCoroutineScope()
    var selectedItem by remember { mutableIntStateOf(0) }

    val items = listOf(
        "ホーム" to Icons.Default.Home,
        "お気に入り" to Icons.Default.Favorite,
        "設定" to Icons.Default.Settings,
        "ヘルプ" to Icons.Default.Help
    )

    ModalNavigationDrawer(
        drawerState = drawerState,
        drawerContent = {
            ModalDrawerSheet {
                Spacer(Modifier.height(16.dp))
                Text(
                    "マイアプリ",
                    modifier = Modifier.padding(16.dp),
                    style = MaterialTheme.typography.titleLarge
                )
                HorizontalDivider(Modifier.padding(horizontal = 16.dp))
                Spacer(Modifier.height(8.dp))

                items.forEachIndexed { index, (label, icon) ->
                    NavigationDrawerItem(
                        icon = { Icon(icon, null) },
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
                        IconButton(onClick = {
                            scope.launch { drawerState.open() }
                        }) {
                            Icon(Icons.Default.Menu, "メニュー")
                        }
                    }
                )
            }
        ) { padding ->
            Box(Modifier.padding(padding).fillMaxSize(), contentAlignment = Alignment.Center) {
                Text("${items[selectedItem].first}ページ")
            }
        }
    }
}
```

---

## PermanentNavigationDrawer（タブレット）

```kotlin
@Composable
fun PermanentDrawerExample() {
    var selectedItem by remember { mutableIntStateOf(0) }
    val items = listOf("受信トレイ", "送信済み", "下書き", "ゴミ箱")

    PermanentNavigationDrawer(
        drawerContent = {
            PermanentDrawerSheet(Modifier.width(240.dp)) {
                Spacer(Modifier.height(16.dp))
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
        // メインコンテンツ
        Scaffold { padding ->
            Box(Modifier.padding(padding).fillMaxSize()) {
                Text("${items[selectedItem]}の内容")
            }
        }
    }
}
```

---

## Adaptive Drawer（画面サイズ対応）

```kotlin
@Composable
fun AdaptiveDrawer() {
    val windowSizeClass = currentWindowAdaptiveInfo().windowSizeClass

    if (windowSizeClass.windowWidthSizeClass == WindowWidthSizeClass.EXPANDED) {
        PermanentDrawerExample()
    } else {
        DrawerExample()
    }
}
```

- タブレット・PC: `PermanentNavigationDrawer`
- スマートフォン: `ModalNavigationDrawer`

---

## ユーザープロフィール付きDrawer

```kotlin
ModalDrawerSheet {
    // ヘッダー
    Box(
        Modifier
            .fillMaxWidth()
            .height(180.dp)
            .background(MaterialTheme.colorScheme.primaryContainer)
            .padding(16.dp),
        contentAlignment = Alignment.BottomStart
    ) {
        Column {
            AsyncImage(
                model = user.avatarUrl,
                contentDescription = null,
                modifier = Modifier
                    .size(64.dp)
                    .clip(CircleShape)
            )
            Spacer(Modifier.height(8.dp))
            Text(user.name, style = MaterialTheme.typography.titleMedium)
            Text(user.email, style = MaterialTheme.typography.bodySmall)
        }
    }

    // メニュー項目
    items.forEach { /* NavigationDrawerItem */ }
}
```

---

## まとめ

- `ModalNavigationDrawer` — スマートフォン向けスライドメニュー
- `PermanentNavigationDrawer` — タブレット向け常駐メニュー
- `rememberDrawerState` + `scope.launch`で開閉制御
- `NavigationDrawerItem`でメニュー項目を配置
- `WindowSizeClass`で画面サイズに応じて切り替え

---

8種類のAndroidアプリテンプレート（ナビゲーション設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-2026)
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-navigation-compose-2026)
- [Adaptive Layout完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-adaptive-layout-2026)
