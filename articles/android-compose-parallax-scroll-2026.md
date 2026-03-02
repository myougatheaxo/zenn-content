---
title: "パララックススクロール完全ガイド — ヘッダー/背景/多層効果"
emoji: "🏔️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**パララックススクロール**（CollapsingHeader、背景パララックス、多層効果、スムーズアニメーション）を解説します。

---

## 基本パララックスヘッダー

```kotlin
@Composable
fun ParallaxHeader() {
    val scrollState = rememberScrollState()
    val headerHeight = 300.dp
    val headerHeightPx = with(LocalDensity.current) { headerHeight.toPx() }

    Box(Modifier.fillMaxSize()) {
        // パララックス背景（スクロールの半分の速度で移動）
        Image(
            painter = painterResource(R.drawable.header_bg),
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(headerHeight)
                .graphicsLayer {
                    translationY = scrollState.value * 0.5f
                    alpha = 1f - (scrollState.value / headerHeightPx).coerceIn(0f, 1f)
                },
            contentScale = ContentScale.Crop
        )

        // コンテンツ
        Column(
            Modifier
                .verticalScroll(scrollState)
                .padding(top = headerHeight)
        ) {
            repeat(30) { index ->
                ListItem(
                    headlineContent = { Text("Item $index") },
                    modifier = Modifier.padding(horizontal = 16.dp)
                )
            }
        }
    }
}
```

---

## LazyColumn + パララックス

```kotlin
@Composable
fun ParallaxLazyColumn() {
    val lazyListState = rememberLazyListState()

    LazyColumn(state = lazyListState) {
        // パララックスヘッダー
        item {
            val firstVisibleItem = lazyListState.firstVisibleItemIndex
            val scrollOffset = lazyListState.firstVisibleItemScrollOffset

            val parallaxOffset = if (firstVisibleItem == 0) scrollOffset * 0.5f else 0f

            Box(Modifier.fillMaxWidth().height(250.dp)) {
                Image(
                    painter = painterResource(R.drawable.header),
                    contentDescription = null,
                    modifier = Modifier
                        .fillMaxSize()
                        .graphicsLayer { translationY = parallaxOffset },
                    contentScale = ContentScale.Crop
                )

                // オーバーレイテキスト
                Text(
                    "パララックス",
                    modifier = Modifier
                        .align(Alignment.BottomStart)
                        .padding(16.dp)
                        .graphicsLayer {
                            alpha = 1f - (parallaxOffset / 200f).coerceIn(0f, 1f)
                        },
                    style = MaterialTheme.typography.headlineLarge,
                    color = Color.White
                )
            }
        }

        // リストアイテム
        items(50) { index ->
            ListItem(headlineContent = { Text("Item $index") })
        }
    }
}
```

---

## 多層パララックス

```kotlin
@Composable
fun MultiLayerParallax() {
    val scrollState = rememberScrollState()

    Box(Modifier.fillMaxSize()) {
        // 背景層（最も遅く移動）
        Image(
            painter = painterResource(R.drawable.layer_bg),
            contentDescription = null,
            modifier = Modifier
                .fillMaxSize()
                .graphicsLayer { translationY = scrollState.value * 0.2f },
            contentScale = ContentScale.Crop
        )

        // 中間層
        Image(
            painter = painterResource(R.drawable.layer_mid),
            contentDescription = null,
            modifier = Modifier
                .fillMaxSize()
                .graphicsLayer { translationY = scrollState.value * 0.5f },
            contentScale = ContentScale.Crop
        )

        // 前景コンテンツ（通常速度）
        Column(Modifier.verticalScroll(scrollState)) {
            Spacer(Modifier.height(400.dp))
            Surface(
                Modifier.fillMaxWidth(),
                color = MaterialTheme.colorScheme.surface,
                shape = RoundedCornerShape(topStart = 24.dp, topEnd = 24.dp)
            ) {
                Column(Modifier.padding(16.dp)) {
                    repeat(20) {
                        Text("コンテンツ $it", Modifier.padding(vertical = 8.dp))
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| 層 | 速度 | 効果 |
|----|------|------|
| 背景 | 0.2x | 遠くに見える |
| 中間 | 0.5x | 中間距離 |
| 前景 | 1.0x | 通常スクロール |

- `graphicsLayer { translationY }`でパララックス効果
- スクロール速度の倍率で奥行き感を表現
- `alpha`変化でフェードアウト効果
- `LazyColumn`ではfirstVisibleItemScrollOffsetを使用

---

8種類のAndroidアプリテンプレート（リッチUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-optimization-2026)
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-custom-drawing-2026)
