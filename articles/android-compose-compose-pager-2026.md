---
title: "Pager完全ガイド — HorizontalPager/VerticalPager/インジケーター/変換"
emoji: "📖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Pager**（HorizontalPager、VerticalPager、ページインジケーター、ページ変換エフェクト）を解説します。

---

## HorizontalPager

```kotlin
@Composable
fun OnboardingPager() {
    val pages = listOf(
        "ようこそ" to "アプリの紹介",
        "機能" to "便利な機能を紹介",
        "始めましょう" to "さっそく使ってみよう"
    )
    val pagerState = rememberPagerState(pageCount = { pages.size })

    Column(Modifier.fillMaxSize()) {
        HorizontalPager(
            state = pagerState,
            modifier = Modifier.weight(1f)
        ) { page ->
            Column(
                Modifier.fillMaxSize().padding(32.dp),
                verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text(pages[page].first, style = MaterialTheme.typography.headlineLarge)
                Spacer(Modifier.height(16.dp))
                Text(pages[page].second, style = MaterialTheme.typography.bodyLarge)
            }
        }

        // ドットインジケーター
        Row(
            Modifier.fillMaxWidth().padding(16.dp),
            horizontalArrangement = Arrangement.Center
        ) {
            repeat(pages.size) { index ->
                Box(
                    Modifier
                        .padding(4.dp)
                        .size(if (pagerState.currentPage == index) 12.dp else 8.dp)
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

## VerticalPager

```kotlin
@Composable
fun VerticalCardPager(items: List<CardItem>) {
    val pagerState = rememberPagerState(pageCount = { items.size })

    VerticalPager(
        state = pagerState,
        modifier = Modifier.fillMaxSize(),
        pageSpacing = 16.dp,
        contentPadding = PaddingValues(vertical = 32.dp)
    ) { page ->
        Card(
            Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp)
                .graphicsLayer {
                    val pageOffset = (pagerState.currentPage - page) +
                        pagerState.currentPageOffsetFraction
                    alpha = lerp(0.5f, 1f, 1f - pageOffset.absoluteValue.coerceIn(0f, 1f))
                    scaleX = lerp(0.85f, 1f, 1f - pageOffset.absoluteValue.coerceIn(0f, 1f))
                    scaleY = lerp(0.85f, 1f, 1f - pageOffset.absoluteValue.coerceIn(0f, 1f))
                }
        ) {
            Column(Modifier.padding(24.dp)) {
                Text(items[page].title, style = MaterialTheme.typography.headlineSmall)
                Spacer(Modifier.height(8.dp))
                Text(items[page].description)
            }
        }
    }
}
```

---

## ページ変更検知

```kotlin
@Composable
fun PagerWithCallback() {
    val pagerState = rememberPagerState(pageCount = { 5 })

    LaunchedEffect(pagerState) {
        snapshotFlow { pagerState.currentPage }.collect { page ->
            // ページ変更時の処理（Analytics等）
            Log.d("Pager", "Current page: $page")
        }
    }

    HorizontalPager(state = pagerState) { page ->
        Text("Page $page", Modifier.fillMaxSize().padding(16.dp))
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `HorizontalPager` | 横スワイプページ |
| `VerticalPager` | 縦スワイプページ |
| `PagerState` | ページ状態管理 |
| `currentPageOffsetFraction` | ページ変換エフェクト |

- `HorizontalPager`でオンボーディング/画像ギャラリー
- `VerticalPager`でTikTok風縦スクロール
- `graphicsLayer`でスケール/フェードエフェクト
- `snapshotFlow`でページ変更を検知

---

8種類のAndroidアプリテンプレート（Pager対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [TabLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-tab-layout-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-2026)
