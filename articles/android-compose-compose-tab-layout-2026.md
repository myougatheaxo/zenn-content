---
title: "TabLayout完全ガイド — TabRow/ScrollableTabRow/Pager連携/バッジ"
emoji: "📑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**TabLayout**（TabRow、ScrollableTabRow、HorizontalPager連携、バッジ付きタブ）を解説します。

---

## 基本TabRow

```kotlin
@Composable
fun TabRowExample() {
    var selectedTab by remember { mutableIntStateOf(0) }
    val tabs = listOf("ホーム", "検索", "設定")

    Column {
        TabRow(selectedTabIndex = selectedTab) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = selectedTab == index,
                    onClick = { selectedTab = index },
                    text = { Text(title) },
                    icon = {
                        Icon(
                            when (index) {
                                0 -> Icons.Default.Home
                                1 -> Icons.Default.Search
                                else -> Icons.Default.Settings
                            }, null
                        )
                    }
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

## HorizontalPager連携

```kotlin
@Composable
fun TabPagerExample() {
    val tabs = listOf("記事", "動画", "ブックマーク")
    val pagerState = rememberPagerState(pageCount = { tabs.size })
    val scope = rememberCoroutineScope()

    Column {
        ScrollableTabRow(
            selectedTabIndex = pagerState.currentPage,
            edgePadding = 16.dp
        ) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = pagerState.currentPage == index,
                    onClick = { scope.launch { pagerState.animateScrollToPage(index) } },
                    text = { Text(title) }
                )
            }
        }

        HorizontalPager(state = pagerState) { page ->
            Box(Modifier.fillMaxSize().padding(16.dp)) {
                Text("${tabs[page]}のコンテンツ", style = MaterialTheme.typography.headlineMedium)
            }
        }
    }
}
```

---

## バッジ付きタブ

```kotlin
@Composable
fun BadgeTabExample() {
    var selectedTab by remember { mutableIntStateOf(0) }
    val badges = listOf(3, 0, 1) // 未読数

    TabRow(selectedTabIndex = selectedTab) {
        listOf("通知", "メッセージ", "更新").forEachIndexed { index, title ->
            Tab(
                selected = selectedTab == index,
                onClick = { selectedTab = index },
                text = { Text(title) },
                icon = {
                    BadgedBox(
                        badge = {
                            if (badges[index] > 0) {
                                Badge { Text("${badges[index]}") }
                            }
                        }
                    ) {
                        Icon(
                            when (index) {
                                0 -> Icons.Default.Notifications
                                1 -> Icons.Default.Email
                                else -> Icons.Default.Update
                            }, null
                        )
                    }
                }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `TabRow` | 固定タブ |
| `ScrollableTabRow` | スクロール可能タブ |
| `HorizontalPager` | スワイプ切り替え |
| `BadgedBox` | バッジ表示 |

- `TabRow`で固定数タブ、`ScrollableTabRow`で多数タブ
- `HorizontalPager`でスワイプによるタブ切り替え
- `BadgedBox`で未読数バッジ表示
- `animateScrollToPage`でプログラム的にページ遷移

---

8種類のAndroidアプリテンプレート（ナビゲーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Bottom Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
- [Pager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pager-2026)
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
