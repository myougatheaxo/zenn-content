---
title: "ピンチズーム完全ガイド — 画像拡大/パン/ダブルタップ/制限"
emoji: "🔎"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**ピンチズーム**（画像拡大縮小、パン移動、ダブルタップズーム、拡大制限、リセットアニメーション）を解説します。

---

## 基本ピンチズーム

```kotlin
@Composable
fun ZoomableImage(painter: Painter) {
    var scale by remember { mutableFloatStateOf(1f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    Image(
        painter = painter,
        contentDescription = null,
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTransformGestures { _, pan, zoom, _ ->
                    scale = (scale * zoom).coerceIn(0.5f, 5f)
                    offset += pan
                }
            }
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                translationX = offset.x
                translationY = offset.y
            },
        contentScale = ContentScale.Fit
    )
}
```

---

## ダブルタップ + ピンチズーム

```kotlin
@Composable
fun AdvancedZoomableImage(painter: Painter) {
    var scale by remember { mutableFloatStateOf(1f) }
    var offset by remember { mutableStateOf(Offset.Zero) }
    val scope = rememberCoroutineScope()
    val animatedScale = remember { Animatable(1f) }
    val animatedOffsetX = remember { Animatable(0f) }
    val animatedOffsetY = remember { Animatable(0f) }

    Image(
        painter = painter,
        contentDescription = null,
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTransformGestures { _, pan, zoom, _ ->
                    scale = (scale * zoom).coerceIn(1f, 5f)
                    if (scale > 1f) {
                        offset += pan
                    }
                }
            }
            .pointerInput(Unit) {
                detectTapGestures(
                    onDoubleTap = { tapOffset ->
                        scope.launch {
                            if (scale > 1.5f) {
                                // リセット
                                launch { animatedScale.animateTo(1f, spring()) }
                                launch { animatedOffsetX.animateTo(0f, spring()) }
                                launch { animatedOffsetY.animateTo(0f, spring()) }
                                scale = 1f; offset = Offset.Zero
                            } else {
                                // ズームイン
                                scale = 3f
                                offset = Offset(
                                    (size.width / 2f - tapOffset.x) * 2f,
                                    (size.height / 2f - tapOffset.y) * 2f
                                )
                            }
                        }
                    }
                )
            }
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                translationX = offset.x
                translationY = offset.y
            },
        contentScale = ContentScale.Fit
    )
}
```

---

## フルスクリーン画像ビューア

```kotlin
@Composable
fun FullScreenImageViewer(
    images: List<String>,
    initialIndex: Int,
    onDismiss: () -> Unit
) {
    val pagerState = rememberPagerState(initialPage = initialIndex) { images.size }

    Box(
        Modifier
            .fillMaxSize()
            .background(Color.Black)
    ) {
        HorizontalPager(state = pagerState) { page ->
            ZoomableImage(
                painter = rememberAsyncImagePainter(images[page])
            )
        }

        // 閉じるボタン
        IconButton(
            onClick = onDismiss,
            modifier = Modifier
                .align(Alignment.TopEnd)
                .padding(16.dp)
        ) {
            Icon(Icons.Default.Close, "閉じる", tint = Color.White)
        }

        // ページインジケーター
        Text(
            "${pagerState.currentPage + 1} / ${images.size}",
            color = Color.White,
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .padding(16.dp)
        )
    }
}
```

---

## まとめ

| ジェスチャー | 動作 |
|-------------|------|
| ピンチ | 拡大/縮小 |
| パン | 拡大中の移動 |
| ダブルタップ | ズームイン/リセット |
| スワイプ | 画像切替 |

- `detectTransformGestures`でピンチ/パン検出
- `graphicsLayer`でスケール/トランスレーション適用
- ダブルタップで3倍ズーム/リセット切替
- `HorizontalPager`で複数画像のスワイプ表示

---

8種類のAndroidアプリテンプレート（画像ビューア実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-gesture-touch-2026)
- [Coil画像](https://zenn.dev/myougatheaxo/articles/android-compose-coil-image-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
