---
title: "TabLayout + ViewPager in Compose — タブUI実装ガイド"
emoji: "📑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Composeでの**TabLayout + HorizontalPager**（タブ切り替え、スワイプ、スクロール可能タブ）を解説します。

---

## 基本のTabRow

```kotlin
@Composable
fun BasicTabExample() {
    var selectedTab by remember { mutableIntStateOf(0) }
    val tabs = listOf("ホーム", "検索", "設定")

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
            1 -> SearchContent()
            2 -> SettingsContent()
        }
    }
}
```

---

## TabRow + HorizontalPager

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun TabWithPager() {
    val pagerState = rememberPagerState { 3 }
    val scope = rememberCoroutineScope()
    val tabs = listOf("人気", "新着", "フォロー中")

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
                0 -> PopularContent()
                1 -> LatestContent()
                2 -> FollowingContent()
            }
        }
    }
}
```

---

## ScrollableTabRow

```kotlin
@Composable
fun ScrollableTabExample() {
    var selectedTab by remember { mutableIntStateOf(0) }
    val tabs = listOf("全て", "Android", "iOS", "Web", "Backend", "DevOps", "AI/ML")

    Column {
        ScrollableTabRow(
            selectedTabIndex = selectedTab,
            edgePadding = 16.dp
        ) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = selectedTab == index,
                    onClick = { selectedTab = index },
                    text = { Text(title) }
                )
            }
        }

        // コンテンツ
        Box(Modifier.fillMaxSize().padding(16.dp)) {
            Text("${tabs[selectedTab]}の内容")
        }
    }
}
```

---

## アイコン付きタブ

```kotlin
@Composable
fun IconTabExample() {
    var selectedTab by remember { mutableIntStateOf(0) }

    data class TabItem(val title: String, val icon: ImageVector)

    val tabs = listOf(
        TabItem("チャット", Icons.Default.Chat),
        TabItem("通話", Icons.Default.Call),
        TabItem("連絡先", Icons.Default.Contacts)
    )

    TabRow(selectedTabIndex = selectedTab) {
        tabs.forEachIndexed { index, tab ->
            Tab(
                selected = selectedTab == index,
                onClick = { selectedTab = index },
                text = { Text(tab.title) },
                icon = { Icon(tab.icon, contentDescription = tab.title) }
            )
        }
    }
}
```

---

## まとめ

- `TabRow`で固定タブ、`ScrollableTabRow`でスクロール可能タブ
- `HorizontalPager`+`pagerState`でスワイプ連携
- `animateScrollToPage`でタブクリック時のアニメーション
- `Tab`にtext/iconの組み合わせ
- `edgePadding`でScrollableTabRowの余白調整
- カスタムインジケーターで独自デザイン

---

8種類のAndroidアプリテンプレート（タブUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [HorizontalPagerガイド](https://zenn.dev/myougatheaxo/articles/android-compose-horizontal-pager-2026)
- [BottomNavigationガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
