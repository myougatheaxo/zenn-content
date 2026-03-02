---
title: "HorizontalPager完全ガイド — カルーセル/オンボーディング/インジケーター"
emoji: "📖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "pager"]
published: true
---

## この記事で学べること

Composeの**HorizontalPager**（カルーセル、オンボーディング、ページインジケーター）を解説します。

---

## 基本のHorizontalPager

```kotlin
@Composable
fun BasicPager() {
    val pagerState = rememberPagerState(pageCount = { 5 })

    Column {
        HorizontalPager(
            state = pagerState,
            modifier = Modifier
                .fillMaxWidth()
                .height(200.dp)
        ) { page ->
            Card(
                Modifier
                    .fillMaxSize()
                    .padding(horizontal = 16.dp)
            ) {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("ページ ${page + 1}", style = MaterialTheme.typography.headlineMedium)
                }
            }
        }

        // ドットインジケーター
        Row(
            Modifier.fillMaxWidth().padding(8.dp),
            horizontalArrangement = Arrangement.Center
        ) {
            repeat(pagerState.pageCount) { index ->
                val color = if (pagerState.currentPage == index)
                    MaterialTheme.colorScheme.primary
                else MaterialTheme.colorScheme.surfaceVariant
                Box(
                    Modifier
                        .padding(4.dp)
                        .size(8.dp)
                        .clip(CircleShape)
                        .background(color)
                )
            }
        }
    }
}
```

---

## 画像カルーセル

```kotlin
@Composable
fun ImageCarousel(images: List<String>) {
    val pagerState = rememberPagerState(pageCount = { images.size })

    Box {
        HorizontalPager(
            state = pagerState,
            modifier = Modifier
                .fillMaxWidth()
                .height(250.dp)
        ) { page ->
            AsyncImage(
                model = images[page],
                contentDescription = "Image ${page + 1}",
                modifier = Modifier.fillMaxSize(),
                contentScale = ContentScale.Crop
            )
        }

        // ページカウンター
        Surface(
            color = Color.Black.copy(alpha = 0.5f),
            shape = RoundedCornerShape(16.dp),
            modifier = Modifier
                .align(Alignment.BottomEnd)
                .padding(8.dp)
        ) {
            Text(
                "${pagerState.currentPage + 1}/${images.size}",
                color = Color.White,
                modifier = Modifier.padding(horizontal = 12.dp, vertical = 4.dp),
                style = MaterialTheme.typography.labelMedium
            )
        }
    }
}
```

---

## オンボーディング

```kotlin
@Composable
fun OnboardingScreen(onComplete: () -> Unit) {
    val pages = listOf(
        OnboardingPage("ようこそ", "アプリの概要説明", Icons.Default.Rocket),
        OnboardingPage("機能紹介", "主要機能の説明", Icons.Default.Star),
        OnboardingPage("始めましょう", "セットアップ完了", Icons.Default.Check)
    )
    val pagerState = rememberPagerState(pageCount = { pages.size })
    val scope = rememberCoroutineScope()

    Column(Modifier.fillMaxSize()) {
        HorizontalPager(
            state = pagerState,
            modifier = Modifier.weight(1f)
        ) { page ->
            Column(
                Modifier.fillMaxSize().padding(32.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Icon(pages[page].icon, null, Modifier.size(80.dp), tint = MaterialTheme.colorScheme.primary)
                Spacer(Modifier.height(24.dp))
                Text(pages[page].title, style = MaterialTheme.typography.headlineMedium)
                Spacer(Modifier.height(8.dp))
                Text(pages[page].description, textAlign = TextAlign.Center)
            }
        }

        // ナビゲーション
        Row(
            Modifier.fillMaxWidth().padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            if (pagerState.currentPage > 0) {
                TextButton(onClick = {
                    scope.launch { pagerState.animateScrollToPage(pagerState.currentPage - 1) }
                }) { Text("戻る") }
            } else Spacer(Modifier.width(1.dp))

            // ドットインジケーター
            Row {
                repeat(pages.size) { i ->
                    Box(
                        Modifier
                            .padding(4.dp)
                            .size(if (i == pagerState.currentPage) 10.dp else 8.dp)
                            .clip(CircleShape)
                            .background(
                                if (i == pagerState.currentPage) MaterialTheme.colorScheme.primary
                                else MaterialTheme.colorScheme.surfaceVariant
                            )
                    )
                }
            }

            if (pagerState.currentPage < pages.lastIndex) {
                Button(onClick = {
                    scope.launch { pagerState.animateScrollToPage(pagerState.currentPage + 1) }
                }) { Text("次へ") }
            } else {
                Button(onClick = onComplete) { Text("始める") }
            }
        }
    }
}

data class OnboardingPage(val title: String, val description: String, val icon: ImageVector)
```

---

## まとめ

- `HorizontalPager`でスワイプ可能なページ
- `rememberPagerState(pageCount = {})`で状態管理
- `currentPage`でインジケーター表示
- `animateScrollToPage()`でプログラムからページ移動
- 画像カルーセル: AsyncImage + ページカウンター
- オンボーディング: 戻る/次へ/始めるボタン制御

---

8種類のAndroidアプリテンプレート（Pager/オンボーディング設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [HorizontalPager+Tabガイド](https://zenn.dev/myougatheaxo/articles/android-compose-tab-pager-2026)
- [オンボーディング実装](https://zenn.dev/myougatheaxo/articles/android-compose-onboarding-2026)
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
