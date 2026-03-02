---
title: "Placeholder/Shimmer完全ガイド — ローディングUI/スケルトン表示"
emoji: "💀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "loading"]
published: true
---

## この記事で学べること

**Placeholder/Shimmer**（ローディングスケルトン、シマーエフェクト、プレースホルダーUI）を解説します。

---

## シンプルスケルトン

```kotlin
@Composable
fun SkeletonItem(modifier: Modifier = Modifier) {
    val infiniteTransition = rememberInfiniteTransition(label = "shimmer")
    val alpha by infiniteTransition.animateFloat(
        initialValue = 0.3f, targetValue = 0.7f,
        animationSpec = infiniteRepeatable(
            tween(1000), RepeatMode.Reverse
        ), label = "alpha"
    )

    Row(modifier.padding(16.dp)) {
        // アバタースケルトン
        Box(
            Modifier
                .size(48.dp)
                .background(Color.LightGray.copy(alpha = alpha), CircleShape)
        )
        Spacer(Modifier.width(12.dp))
        Column(Modifier.weight(1f)) {
            Box(
                Modifier
                    .fillMaxWidth(0.7f)
                    .height(16.dp)
                    .background(Color.LightGray.copy(alpha = alpha), RoundedCornerShape(4.dp))
            )
            Spacer(Modifier.height(8.dp))
            Box(
                Modifier
                    .fillMaxWidth(0.5f)
                    .height(12.dp)
                    .background(Color.LightGray.copy(alpha = alpha), RoundedCornerShape(4.dp))
            )
        }
    }
}

@Composable
fun SkeletonList(count: Int = 5) {
    LazyColumn {
        items(count) { SkeletonItem(Modifier.fillMaxWidth()) }
    }
}
```

---

## シマーエフェクト

```kotlin
@Composable
fun ShimmerEffect(modifier: Modifier = Modifier) {
    val infiniteTransition = rememberInfiniteTransition(label = "shimmer")
    val offset by infiniteTransition.animateFloat(
        initialValue = -500f, targetValue = 1500f,
        animationSpec = infiniteRepeatable(tween(1200, easing = LinearEasing)),
        label = "offset"
    )

    Box(
        modifier
            .drawWithCache {
                val brush = Brush.linearGradient(
                    listOf(
                        Color.LightGray.copy(alpha = 0.4f),
                        Color.LightGray.copy(alpha = 0.8f),
                        Color.LightGray.copy(alpha = 0.4f)
                    ),
                    start = Offset(offset, 0f),
                    end = Offset(offset + 500f, 0f)
                )
                onDrawBehind { drawRect(brush) }
            }
    )
}

@Composable
fun ShimmerCard() {
    Card(Modifier.fillMaxWidth().padding(16.dp)) {
        Column(Modifier.padding(16.dp)) {
            ShimmerEffect(
                Modifier.fillMaxWidth().height(160.dp)
                    .clip(RoundedCornerShape(8.dp))
            )
            Spacer(Modifier.height(12.dp))
            ShimmerEffect(
                Modifier.fillMaxWidth(0.8f).height(20.dp)
                    .clip(RoundedCornerShape(4.dp))
            )
            Spacer(Modifier.height(8.dp))
            ShimmerEffect(
                Modifier.fillMaxWidth(0.6f).height(14.dp)
                    .clip(RoundedCornerShape(4.dp))
            )
        }
    }
}
```

---

## 条件分岐表示

```kotlin
@Composable
fun UserProfileScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is UiState.Loading -> {
            Column(Modifier.padding(16.dp)) {
                ShimmerEffect(Modifier.size(80.dp).clip(CircleShape))
                Spacer(Modifier.height(16.dp))
                ShimmerEffect(Modifier.fillMaxWidth(0.5f).height(24.dp).clip(RoundedCornerShape(4.dp)))
                Spacer(Modifier.height(8.dp))
                ShimmerEffect(Modifier.fillMaxWidth(0.7f).height(16.dp).clip(RoundedCornerShape(4.dp)))
            }
        }
        is UiState.Success -> {
            val user = (uiState as UiState.Success).user
            Column(Modifier.padding(16.dp)) {
                AsyncImage(model = user.avatarUrl, contentDescription = null, Modifier.size(80.dp).clip(CircleShape))
                Spacer(Modifier.height(16.dp))
                Text(user.name, style = MaterialTheme.typography.headlineSmall)
                Text(user.email, style = MaterialTheme.typography.bodyMedium)
            }
        }
        is UiState.Error -> { Text("エラーが発生しました", Modifier.padding(16.dp)) }
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| スケルトン | 形状のプレースホルダー |
| シマー | 光沢アニメーション |
| `drawWithCache` | パフォーマンス最適化 |
| 条件分岐 | Loading/Success/Error |

- スケルトンUIでローディング中の体験向上
- シマーエフェクトで「読込中」を視覚的に伝達
- `drawWithCache`でBrush再生成を最適化
- UiStateパターンで表示切替

---

8種類のAndroidアプリテンプレート（ローディングUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ProgressIndicator](https://zenn.dev/myougatheaxo/articles/android-compose-compose-progress-indicator-2026)
- [AnimatedVisibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-visibility-2026)
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
