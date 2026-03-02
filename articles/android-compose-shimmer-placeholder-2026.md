---
title: "Shimmer/Placeholder完全ガイド — スケルトンUI/ローディング表示"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Shimmer/Placeholder**（スケルトンUI、シマー効果、プレースホルダー、ローディング状態管理）を解説します。

---

## カスタムShimmer Modifier

```kotlin
fun Modifier.shimmer(
    isLoading: Boolean,
    shimmerColor: Color = Color.LightGray.copy(alpha = 0.6f),
    baseColor: Color = Color.LightGray.copy(alpha = 0.2f)
): Modifier = composed {
    if (!isLoading) return@composed this

    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateX by transition.animateFloat(
        initialValue = -300f,
        targetValue = 300f,
        animationSpec = infiniteRepeatable(
            animation = tween(1200, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmerX"
    )

    this.drawWithContent {
        drawContent()
        val brush = Brush.linearGradient(
            colors = listOf(baseColor, shimmerColor, baseColor),
            start = Offset(translateX, 0f),
            end = Offset(translateX + 300f, 0f)
        )
        drawRect(brush = brush, size = size)
    }
}
```

---

## スケルトンUI

```kotlin
@Composable
fun ArticleListSkeleton(count: Int = 5) {
    LazyColumn(Modifier.fillMaxSize().padding(16.dp)) {
        items(count) {
            ArticleSkeletonItem()
            Spacer(Modifier.height(12.dp))
        }
    }
}

@Composable
fun ArticleSkeletonItem() {
    Card(Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp)) {
            // タイトル行
            Box(
                Modifier
                    .fillMaxWidth(0.8f)
                    .height(20.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmer(true)
            )
            Spacer(Modifier.height(8.dp))
            // 本文行1
            Box(
                Modifier
                    .fillMaxWidth()
                    .height(14.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmer(true)
            )
            Spacer(Modifier.height(4.dp))
            // 本文行2
            Box(
                Modifier
                    .fillMaxWidth(0.6f)
                    .height(14.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmer(true)
            )
            Spacer(Modifier.height(12.dp))
            // アバター + 名前
            Row(verticalAlignment = Alignment.CenterVertically) {
                Box(
                    Modifier
                        .size(32.dp)
                        .clip(CircleShape)
                        .shimmer(true)
                )
                Spacer(Modifier.width(8.dp))
                Box(
                    Modifier
                        .width(100.dp)
                        .height(12.dp)
                        .clip(RoundedCornerShape(4.dp))
                        .shimmer(true)
                )
            }
        }
    }
}
```

---

## 状態切替

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UiState.Loading -> ArticleListSkeleton()
        is UiState.Success -> {
            LazyColumn {
                items(state.data) { article ->
                    ArticleItem(article)
                }
            }
        }
        is UiState.Error -> ErrorContent(state.message) { viewModel.retry() }
    }
}
```

---

## Crossfade遷移

```kotlin
@Composable
fun SmartContent(isLoading: Boolean, content: @Composable () -> Unit) {
    Crossfade(targetState = isLoading, label = "loading") { loading ->
        if (loading) {
            ArticleListSkeleton()
        } else {
            content()
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|----------|------|
| Shimmer | 光沢アニメーション |
| Skeleton | コンテンツ形状のプレースホルダー |
| Crossfade | スムーズな切替 |
| UiState | 状態管理 |

- シマー効果でローディング中の視覚フィードバック
- スケルトンUIで実コンテンツと同じレイアウト
- `Crossfade`でローディング→コンテンツのスムーズ遷移
- `composed`で再利用可能なModifier

---

8種類のAndroidアプリテンプレート（ローディングUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [エラーUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-optimization-2026)
