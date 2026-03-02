---
title: "パララックス完全ガイド — スクロール連動/画像パララックス/CollapsingHeader"
emoji: "🏔️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**パララックス**（スクロール連動エフェクト、画像パララックス、CollapsingHeader）を解説します。

---

## スクロール連動パララックス

```kotlin
@Composable
fun ParallaxHeader() {
    val listState = rememberLazyListState()

    val headerOffset by remember {
        derivedStateOf {
            if (listState.firstVisibleItemIndex == 0) {
                listState.firstVisibleItemScrollOffset * 0.5f
            } else 0f
        }
    }

    LazyColumn(state = listState) {
        item {
            Box(
                Modifier
                    .fillMaxWidth()
                    .height(300.dp)
                    .graphicsLayer {
                        translationY = headerOffset
                        alpha = 1f - (headerOffset / 500f).coerceIn(0f, 1f)
                    }
            ) {
                AsyncImage(
                    model = "https://example.com/hero.jpg",
                    contentDescription = null,
                    modifier = Modifier.fillMaxSize(),
                    contentScale = ContentScale.Crop
                )
                Box(
                    Modifier.fillMaxSize().background(
                        Brush.verticalGradient(
                            listOf(Color.Transparent, Color.Black.copy(alpha = 0.6f))
                        )
                    )
                )
                Text(
                    "タイトル",
                    color = Color.White,
                    style = MaterialTheme.typography.headlineLarge,
                    modifier = Modifier.align(Alignment.BottomStart).padding(16.dp)
                )
            }
        }
        items(30) { index ->
            ListItem(headlineContent = { Text("コンテンツ $index") })
        }
    }
}
```

---

## カードパララックス

```kotlin
@Composable
fun ParallaxCard() {
    var offset by remember { mutableStateOf(Offset.Zero) }

    Card(
        Modifier
            .fillMaxWidth()
            .height(200.dp)
            .pointerInput(Unit) {
                detectDragGestures(
                    onDrag = { change, _ ->
                        offset = Offset(
                            (change.position.x - size.width / 2) / size.width * 10f,
                            (change.position.y - size.height / 2) / size.height * 10f
                        )
                    },
                    onDragEnd = { offset = Offset.Zero }
                )
            }
            .graphicsLayer {
                rotationX = -offset.y
                rotationY = offset.x
                cameraDistance = 12f * density
            }
    ) {
        Box(Modifier.fillMaxSize().background(MaterialTheme.colorScheme.primaryContainer)) {
            Text("3Dカード", Modifier.align(Alignment.Center))
        }
    }
}
```

---

## まとめ

| 技法 | 実装方法 |
|------|---------|
| スクロールパララックス | `firstVisibleItemScrollOffset` |
| フェード | `graphicsLayer { alpha }` |
| 3Dカード | `rotationX/Y + cameraDistance` |
| グラデーションオーバーレイ | `Brush.verticalGradient` |

- スクロール量をパララックス速度の係数で乗算
- `graphicsLayer`でtranslationY/alphaを制御
- `derivedStateOf`でリコンポーズを最小化
- ヒーロー画像やプロフィールヘッダーに最適

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [NestedScroll](https://zenn.dev/myougatheaxo/articles/android-compose-compose-nested-scroll-2026)
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
