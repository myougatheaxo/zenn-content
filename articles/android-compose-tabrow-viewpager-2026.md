---
title: "TabRow/HorizontalPager完全ガイド — スワイプタブ/カスタムインジケーター"
emoji: "📑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**TabRow/HorizontalPager**（スワイプタブ、カスタムインジケーター、ScrollableTabRow、PagerState連携）を解説します。

---

## TabRow + HorizontalPager

```kotlin
@Composable
fun TabPagerScreen() {
    val tabs = listOf("ホーム", "検索", "設定")
    val pagerState = rememberPagerState(pageCount = { tabs.size })
    val scope = rememberCoroutineScope()

    Column(Modifier.fillMaxSize()) {
        TabRow(
            selectedTabIndex = pagerState.currentPage,
            indicator = { tabPositions ->
                TabRowDefaults.SecondaryIndicator(
                    Modifier.tabIndicatorOffset(tabPositions[pagerState.currentPage]),
                    color = MaterialTheme.colorScheme.primary
                )
            }
        ) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = pagerState.currentPage == index,
                    onClick = { scope.launch { pagerState.animateScrollToPage(index) } },
                    text = { Text(title) }
                )
            }
        }

        HorizontalPager(
            state = pagerState,
            modifier = Modifier.fillMaxSize()
        ) { page ->
            when (page) {
                0 -> HomeTab()
                1 -> SearchTab()
                2 -> SettingsTab()
            }
        }
    }
}
```

---

## ScrollableTabRow

```kotlin
@Composable
fun ScrollableTabExample(categories: List<String>) {
    var selectedIndex by remember { mutableIntStateOf(0) }

    ScrollableTabRow(
        selectedTabIndex = selectedIndex,
        edgePadding = 16.dp,
        divider = {},
        indicator = { tabPositions ->
            Box(
                Modifier
                    .tabIndicatorOffset(tabPositions[selectedIndex])
                    .height(3.dp)
                    .clip(RoundedCornerShape(topStart = 3.dp, topEnd = 3.dp))
                    .background(MaterialTheme.colorScheme.primary)
            )
        }
    ) {
        categories.forEachIndexed { index, category ->
            Tab(
                selected = selectedIndex == index,
                onClick = { selectedIndex = index },
                text = {
                    Text(
                        category,
                        fontWeight = if (selectedIndex == index) FontWeight.Bold else FontWeight.Normal
                    )
                }
            )
        }
    }
}
```

---

## カスタムインジケーター

```kotlin
@Composable
fun PillIndicatorTabs() {
    val tabs = listOf("全て", "未読", "お気に入り")
    var selected by remember { mutableIntStateOf(0) }

    Row(
        Modifier
            .fillMaxWidth()
            .padding(16.dp)
            .background(MaterialTheme.colorScheme.surfaceVariant, RoundedCornerShape(25.dp))
            .padding(4.dp),
        horizontalArrangement = Arrangement.SpaceEvenly
    ) {
        tabs.forEachIndexed { index, title ->
            val isSelected = selected == index
            val bgColor by animateColorAsState(
                if (isSelected) MaterialTheme.colorScheme.primary else Color.Transparent,
                label = "tabBg"
            )

            Box(
                Modifier
                    .weight(1f)
                    .clip(RoundedCornerShape(20.dp))
                    .background(bgColor)
                    .clickable { selected = index }
                    .padding(vertical = 8.dp),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    title,
                    color = if (isSelected)
                        MaterialTheme.colorScheme.onPrimary
                    else MaterialTheme.colorScheme.onSurfaceVariant,
                    fontWeight = if (isSelected) FontWeight.Bold else FontWeight.Normal
                )
            }
        }
    }
}
```

---

## PagerState連携

```kotlin
@Composable
fun PagerWithDots() {
    val pagerState = rememberPagerState(pageCount = { 5 })

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        HorizontalPager(state = pagerState, modifier = Modifier.weight(1f)) { page ->
            OnboardingPage(page)
        }

        // ドットインジケーター
        Row(Modifier.padding(16.dp), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            repeat(5) { index ->
                val width by animateDpAsState(
                    if (pagerState.currentPage == index) 24.dp else 8.dp,
                    label = "dotWidth"
                )
                Box(
                    Modifier
                        .height(8.dp)
                        .width(width)
                        .clip(CircleShape)
                        .background(
                            if (pagerState.currentPage == index)
                                MaterialTheme.colorScheme.primary
                            else MaterialTheme.colorScheme.outline
                        )
                )
            }
        }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|---------------|------|
| `TabRow` | 固定タブ |
| `ScrollableTabRow` | スクロール可能タブ |
| `HorizontalPager` | スワイプページ |
| `PagerState` | ページ状態管理 |

- `HorizontalPager` + `TabRow`でスワイプタブ
- `animateScrollToPage`でプログラム的ページ遷移
- カスタムインジケーターでデザイン自由度
- ドットインジケーターでオンボーディングUI

---

8種類のAndroidアプリテンプレート（タブUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
