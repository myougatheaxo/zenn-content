---
title: "TabLayoutガイド — ScrollableTabRow/PrimaryTabRow実装"
emoji: "📑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "tab"]
published: true
---

## この記事で学べること

Composeの**TabRow/ScrollableTabRow**とHorizontalPagerの連携を解説します。

---

## 基本のTabRow

```kotlin
@Composable
fun BasicTabs() {
    var selectedTab by remember { mutableIntStateOf(0) }
    val tabs = listOf("ホーム", "人気", "新着")

    Column {
        TabRow(selectedTabIndex = selectedTab) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = selectedTab == index,
                    onClick = { selectedTab = index },
                    text = { Text(title) }
                )
            }
        }

        when (selectedTab) {
            0 -> HomeContent()
            1 -> PopularContent()
            2 -> NewContent()
        }
    }
}
```

---

## アイコン付きTab

```kotlin
@Composable
fun IconTabs() {
    var selectedTab by remember { mutableIntStateOf(0) }

    TabRow(selectedTabIndex = selectedTab) {
        Tab(
            selected = selectedTab == 0,
            onClick = { selectedTab = 0 },
            text = { Text("チャット") },
            icon = { Icon(Icons.Default.Chat, null) }
        )
        Tab(
            selected = selectedTab == 1,
            onClick = { selectedTab = 1 },
            text = { Text("通話") },
            icon = {
                BadgedBox(badge = { Badge { Text("3") } }) {
                    Icon(Icons.Default.Call, null)
                }
            }
        )
        Tab(
            selected = selectedTab == 2,
            onClick = { selectedTab = 2 },
            text = { Text("連絡先") },
            icon = { Icon(Icons.Default.Contacts, null) }
        )
    }
}
```

---

## ScrollableTabRow

```kotlin
@Composable
fun ScrollableTabs(categories: List<String>) {
    var selectedTab by remember { mutableIntStateOf(0) }

    ScrollableTabRow(
        selectedTabIndex = selectedTab,
        edgePadding = 16.dp
    ) {
        categories.forEachIndexed { index, category ->
            Tab(
                selected = selectedTab == index,
                onClick = { selectedTab = index },
                text = { Text(category) }
            )
        }
    }
}
```

---

## TabRow + HorizontalPager

```kotlin
@Composable
fun SwipeableTabs() {
    val tabs = listOf("すべて", "未読", "お気に入り")
    val pagerState = rememberPagerState(pageCount = { tabs.size })
    val scope = rememberCoroutineScope()

    Column {
        TabRow(selectedTabIndex = pagerState.currentPage) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = pagerState.currentPage == index,
                    onClick = {
                        scope.launch { pagerState.animateScrollToPage(index) }
                    },
                    text = { Text(title) }
                )
            }
        }

        HorizontalPager(
            state = pagerState,
            modifier = Modifier.fillMaxSize()
        ) { page ->
            when (page) {
                0 -> AllItemsList()
                1 -> UnreadItemsList()
                2 -> FavoriteItemsList()
            }
        }
    }
}
```

---

## カスタムインジケーター

```kotlin
@Composable
fun CustomIndicatorTabs() {
    var selectedTab by remember { mutableIntStateOf(0) }
    val tabs = listOf("タブ1", "タブ2", "タブ3")

    TabRow(
        selectedTabIndex = selectedTab,
        indicator = { tabPositions ->
            Box(
                Modifier
                    .tabIndicatorOffset(tabPositions[selectedTab])
                    .height(4.dp)
                    .padding(horizontal = 24.dp)
                    .clip(RoundedCornerShape(topStart = 4.dp, topEnd = 4.dp))
                    .background(MaterialTheme.colorScheme.primary)
            )
        },
        divider = {} // デフォルトのdividerを消す
    ) {
        tabs.forEachIndexed { index, title ->
            Tab(
                selected = selectedTab == index,
                onClick = { selectedTab = index },
                text = {
                    Text(
                        title,
                        fontWeight = if (selectedTab == index) FontWeight.Bold else FontWeight.Normal
                    )
                }
            )
        }
    }
}
```

---

## まとめ

- `TabRow`: 固定タブ（3-5個推奨）
- `ScrollableTabRow`: 多数のタブ（スクロール可能）
- `HorizontalPager` + `TabRow`でスワイプ切り替え
- `Tab`のicon/textでアイコン+テキスト
- `BadgedBox`でバッジ付きタブ
- `indicator`パラメータでカスタムインジケーター

---

8種類のAndroidアプリテンプレート（TabLayout設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [HorizontalPager実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-tab-pager-2026)
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-basic-2026)
- [Material3コンポーネント一覧](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
