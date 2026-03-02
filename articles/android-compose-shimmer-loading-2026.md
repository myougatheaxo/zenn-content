---
title: "Shimmer Loading効果 — Composeでスケルトンスクリーンを実装"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

データ読み込み中に表示する**Shimmer（スケルトンスクリーン）**エフェクトをComposeで実装する方法を解説します。

---

## Shimmer Modifier

```kotlin
fun Modifier.shimmerEffect(): Modifier = composed {
    var size by remember { mutableStateOf(IntSize.Zero) }
    val transition = rememberInfiniteTransition(label = "shimmer")

    val startOffsetX by transition.animateFloat(
        initialValue = -2 * size.width.toFloat(),
        targetValue = 2 * size.width.toFloat(),
        animationSpec = infiniteRepeatable(
            animation = tween(1000)
        ),
        label = "shimmer"
    )

    background(
        brush = Brush.linearGradient(
            colors = listOf(
                Color(0xFFE0E0E0),
                Color(0xFFF5F5F5),
                Color(0xFFE0E0E0)
            ),
            start = Offset(startOffsetX, 0f),
            end = Offset(startOffsetX + size.width.toFloat(), size.height.toFloat())
        )
    ).onGloballyPositioned {
        size = it.size
    }
}
```

---

## スケルトンカード

```kotlin
@Composable
fun ShimmerCard() {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        shape = RoundedCornerShape(12.dp)
    ) {
        Column(Modifier.padding(16.dp)) {
            // 画像プレースホルダー
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(180.dp)
                    .clip(RoundedCornerShape(8.dp))
                    .shimmerEffect()
            )
            Spacer(Modifier.height(12.dp))

            // タイトルプレースホルダー
            Box(
                modifier = Modifier
                    .fillMaxWidth(0.7f)
                    .height(20.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmerEffect()
            )
            Spacer(Modifier.height(8.dp))

            // サブタイトル
            Box(
                modifier = Modifier
                    .fillMaxWidth(0.5f)
                    .height(16.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmerEffect()
            )
        }
    }
}
```

---

## リストスケルトン

```kotlin
@Composable
fun ShimmerList(itemCount: Int = 5) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(itemCount) {
            ShimmerListItem()
        }
    }
}

@Composable
fun ShimmerListItem() {
    Row(
        Modifier
            .fillMaxWidth()
            .padding(8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // アバター
        Box(
            Modifier
                .size(48.dp)
                .clip(CircleShape)
                .shimmerEffect()
        )
        Spacer(Modifier.width(12.dp))
        Column(Modifier.weight(1f)) {
            Box(
                Modifier
                    .fillMaxWidth(0.6f)
                    .height(16.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmerEffect()
            )
            Spacer(Modifier.height(6.dp))
            Box(
                Modifier
                    .fillMaxWidth(0.4f)
                    .height(12.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmerEffect()
            )
        }
    }
}
```

---

## UiStateと組み合わせ

```kotlin
@Composable
fun ArticleScreen(viewModel: ArticleViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UiState.Loading -> ShimmerList()
        is UiState.Success -> {
            LazyColumn {
                items(state.data) { article ->
                    ArticleItem(article)
                }
            }
        }
        is UiState.Error -> ErrorView(state.message)
    }
}
```

---

## Crossfade遷移

```kotlin
@Composable
fun SmoothLoadingTransition(isLoading: Boolean, content: @Composable () -> Unit) {
    Crossfade(targetState = isLoading, label = "loading") { loading ->
        if (loading) {
            ShimmerList()
        } else {
            content()
        }
    }
}
```

`Crossfade`でShimmerから実コンテンツへの切り替えを滑らかに。

---

## まとめ

- `Modifier.shimmerEffect()`でカスタムShimmer
- `infiniteRepeatable` + `linearGradient`でアニメーション
- `Box` + `clip`でプレースホルダー形状を定義
- `UiState.Loading`時にShimmerを表示
- `Crossfade`でシームレスな遷移

---

8種類のAndroidアプリテンプレート（ローディングUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [状態管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
