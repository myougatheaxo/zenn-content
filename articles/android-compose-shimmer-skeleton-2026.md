---
title: "Shimmer/Skeletonローディング実装ガイド — Compose版"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Composeでの**Shimmer/Skeletonローディング**（光沢アニメーション、プレースホルダー）を解説します。

---

## Shimmerエフェクト

```kotlin
@Composable
fun Modifier.shimmerEffect(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateAnim by transition.animateFloat(
        initialValue = 0f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(1200, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmer"
    )

    val brush = Brush.linearGradient(
        colors = listOf(
            Color.LightGray.copy(alpha = 0.6f),
            Color.LightGray.copy(alpha = 0.2f),
            Color.LightGray.copy(alpha = 0.6f)
        ),
        start = Offset(translateAnim - 200f, translateAnim - 200f),
        end = Offset(translateAnim, translateAnim)
    )

    background(brush)
}
```

---

## Skeletonコンポーネント

```kotlin
@Composable
fun SkeletonListItem() {
    Row(
        Modifier
            .fillMaxWidth()
            .padding(16.dp)
    ) {
        // アバター
        Box(
            Modifier
                .size(48.dp)
                .clip(CircleShape)
                .shimmerEffect()
        )

        Spacer(Modifier.width(16.dp))

        Column(Modifier.weight(1f)) {
            // タイトル
            Box(
                Modifier
                    .fillMaxWidth(0.7f)
                    .height(16.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmerEffect()
            )
            Spacer(Modifier.height(8.dp))
            // サブタイトル
            Box(
                Modifier
                    .fillMaxWidth(0.5f)
                    .height(12.dp)
                    .clip(RoundedCornerShape(4.dp))
                    .shimmerEffect()
            )
        }
    }
}

@Composable
fun SkeletonCard() {
    Card(
        Modifier
            .fillMaxWidth()
            .padding(16.dp)
    ) {
        Column {
            Box(
                Modifier
                    .fillMaxWidth()
                    .height(180.dp)
                    .shimmerEffect()
            )
            Column(Modifier.padding(16.dp)) {
                Box(
                    Modifier
                        .fillMaxWidth(0.8f)
                        .height(20.dp)
                        .clip(RoundedCornerShape(4.dp))
                        .shimmerEffect()
                )
                Spacer(Modifier.height(8.dp))
                Box(
                    Modifier
                        .fillMaxWidth()
                        .height(14.dp)
                        .clip(RoundedCornerShape(4.dp))
                        .shimmerEffect()
                )
            }
        }
    }
}
```

---

## ローディング状態管理

```kotlin
@Composable
fun ContentWithSkeleton(viewModel: ListViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Crossfade(
        targetState = uiState,
        label = "content"
    ) { state ->
        when (state) {
            is UiState.Loading -> {
                LazyColumn {
                    items(5) { SkeletonListItem() }
                }
            }
            is UiState.Success -> {
                LazyColumn {
                    items(state.items) { item ->
                        ActualListItem(item)
                    }
                }
            }
            is UiState.Error -> {
                ErrorScreen(state.message)
            }
        }
    }
}
```

---

## まとめ

- `infiniteTransition`+`linearGradient`でShimmerアニメーション
- Skeleton = 実際のレイアウトと同じ形状のプレースホルダー
- `Crossfade`でローディング→コンテンツの滑らかな遷移
- `Modifier.composed`でステートフルなModifier
- リストアイテム/カード等のUI形状に合わせたSkeleton設計
- `shimmerEffect()`を共通Modifierとして再利用

---

8種類のAndroidアプリテンプレート（ローディングUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ローディング状態管理](https://zenn.dev/myougatheaxo/articles/android-compose-loading-state-2026)
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
- [Pull-to-Refresh](https://zenn.dev/myougatheaxo/articles/android-compose-pull-to-refresh-advanced-2026)
