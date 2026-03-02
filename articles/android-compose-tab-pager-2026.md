---
title: "Compose TabRow + HorizontalPager — スワイプ対応タブUIの実装"
emoji: "📑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**TabRow + HorizontalPager**を組み合わせて、スワイプでタブを切り替えるUIを実装する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.compose.foundation:foundation:1.7.0")
}
```

---

## 基本実装

```kotlin
@Composable
fun TabPagerScreen() {
    val tabs = listOf("すべて", "未完了", "完了")
    val pagerState = rememberPagerState(pageCount = { tabs.size })
    val scope = rememberCoroutineScope()

    Column {
        TabRow(selectedTabIndex = pagerState.currentPage) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = pagerState.currentPage == index,
                    onClick = {
                        scope.launch {
                            pagerState.animateScrollToPage(index)
                        }
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
                0 -> AllTasksPage()
                1 -> PendingTasksPage()
                2 -> CompletedTasksPage()
            }
        }
    }
}
```

---

## アイコン付きタブ

```kotlin
data class TabItem(
    val title: String,
    val selectedIcon: ImageVector,
    val unselectedIcon: ImageVector
)

val tabItems = listOf(
    TabItem("ホーム", Icons.Filled.Home, Icons.Outlined.Home),
    TabItem("お気に入り", Icons.Filled.Favorite, Icons.Outlined.FavoriteBorder),
    TabItem("設定", Icons.Filled.Settings, Icons.Outlined.Settings)
)

TabRow(selectedTabIndex = pagerState.currentPage) {
    tabItems.forEachIndexed { index, item ->
        Tab(
            selected = pagerState.currentPage == index,
            onClick = { scope.launch { pagerState.animateScrollToPage(index) } },
            text = { Text(item.title) },
            icon = {
                Icon(
                    if (pagerState.currentPage == index) item.selectedIcon
                    else item.unselectedIcon,
                    contentDescription = item.title
                )
            }
        )
    }
}
```

---

## ScrollableTabRow（タブが多い場合）

```kotlin
val categories = listOf("テクノロジー", "ビジネス", "デザイン", "マーケティング", "エンジニアリング", "サイエンス")

ScrollableTabRow(
    selectedTabIndex = pagerState.currentPage,
    edgePadding = 16.dp
) {
    categories.forEachIndexed { index, category ->
        Tab(
            selected = pagerState.currentPage == index,
            onClick = { scope.launch { pagerState.animateScrollToPage(index) } },
            text = { Text(category) }
        )
    }
}
```

タブが画面幅に収まらないとき、`ScrollableTabRow`で横スクロール可能に。

---

## タブにバッジ

```kotlin
Tab(
    selected = pagerState.currentPage == index,
    onClick = { /* ... */ },
    text = {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text(tab.title)
            if (tab.badgeCount > 0) {
                Spacer(Modifier.width(4.dp))
                Badge { Text("${tab.badgeCount}") }
            }
        }
    }
)
```

---

## まとめ

- `TabRow` + `HorizontalPager`でスワイプ対応タブ
- `rememberPagerState`でページ状態管理
- `animateScrollToPage`でタップ時のアニメーション遷移
- `ScrollableTabRow`でタブが多い場合に対応
- アイコン・バッジでリッチなタブUI

---

8種類のAndroidアプリテンプレート（タブUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
