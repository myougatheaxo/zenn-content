---
title: "Shimmerエフェクト完全ガイド — ローディング/スケルトン/カスタムアニメーション"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**Shimmerエフェクト**（ローディングアニメーション、カスタムスケルトン、Modifier拡張）を解説します。

---

## Shimmer Modifier

```kotlin
fun Modifier.shimmer(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateAnim by transition.animateFloat(
        initialValue = -1000f, targetValue = 1000f,
        animationSpec = infiniteRepeatable(tween(1200, easing = LinearEasing)),
        label = "translate"
    )

    this.drawWithCache {
        val brush = Brush.linearGradient(
            colors = listOf(
                Color.LightGray.copy(alpha = 0.3f),
                Color.LightGray.copy(alpha = 0.8f),
                Color.LightGray.copy(alpha = 0.3f)
            ),
            start = Offset(translateAnim, 0f),
            end = Offset(translateAnim + 500f, 500f)
        )
        onDrawBehind { drawRect(brush) }
    }
}

@Composable
fun ShimmerUsage() {
    Column(Modifier.padding(16.dp)) {
        Box(Modifier.fillMaxWidth().height(200.dp).clip(RoundedCornerShape(12.dp)).shimmer())
        Spacer(Modifier.height(12.dp))
        Box(Modifier.fillMaxWidth(0.7f).height(20.dp).clip(RoundedCornerShape(4.dp)).shimmer())
        Spacer(Modifier.height(8.dp))
        Box(Modifier.fillMaxWidth(0.5f).height(16.dp).clip(RoundedCornerShape(4.dp)).shimmer())
    }
}
```

---

## リストスケルトン

```kotlin
@Composable
fun ShimmerListItem(modifier: Modifier = Modifier) {
    Row(modifier.padding(12.dp), verticalAlignment = Alignment.CenterVertically) {
        Box(Modifier.size(56.dp).clip(CircleShape).shimmer())
        Spacer(Modifier.width(12.dp))
        Column(Modifier.weight(1f)) {
            Box(Modifier.fillMaxWidth(0.8f).height(16.dp).clip(RoundedCornerShape(4.dp)).shimmer())
            Spacer(Modifier.height(6.dp))
            Box(Modifier.fillMaxWidth(0.5f).height(12.dp).clip(RoundedCornerShape(4.dp)).shimmer())
        }
    }
}

@Composable
fun LoadingList(isLoading: Boolean, items: List<String>, content: @Composable (String) -> Unit) {
    LazyColumn {
        if (isLoading) {
            items(6) { ShimmerListItem(Modifier.fillMaxWidth()) }
        } else {
            items(items) { content(it) }
        }
    }
}
```

---

## カードスケルトン

```kotlin
@Composable
fun ShimmerCard(modifier: Modifier = Modifier) {
    Card(modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp)) {
            Row {
                Box(Modifier.size(40.dp).clip(CircleShape).shimmer())
                Spacer(Modifier.width(8.dp))
                Column {
                    Box(Modifier.width(100.dp).height(14.dp).clip(RoundedCornerShape(4.dp)).shimmer())
                    Spacer(Modifier.height(4.dp))
                    Box(Modifier.width(60.dp).height(10.dp).clip(RoundedCornerShape(4.dp)).shimmer())
                }
            }
            Spacer(Modifier.height(12.dp))
            Box(Modifier.fillMaxWidth().height(160.dp).clip(RoundedCornerShape(8.dp)).shimmer())
            Spacer(Modifier.height(8.dp))
            Box(Modifier.fillMaxWidth(0.9f).height(14.dp).clip(RoundedCornerShape(4.dp)).shimmer())
            Spacer(Modifier.height(4.dp))
            Box(Modifier.fillMaxWidth(0.7f).height(14.dp).clip(RoundedCornerShape(4.dp)).shimmer())
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `Modifier.shimmer()` | 再利用可能なシマー |
| `drawWithCache` | 描画キャッシュ最適化 |
| `composed` | Modifier内でCompose使用 |
| `clip + shimmer` | 形状に合わせたシマー |

- `Modifier.shimmer()`でどの要素にもシマー適用
- `drawWithCache`でBrush再生成を最適化
- `clip`で丸/角丸などの形状に合わせて表示
- ローディング中のUXを大幅に向上

---

8種類のAndroidアプリテンプレート（ローディングUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Placeholder/スケルトン](https://zenn.dev/myougatheaxo/articles/android-compose-compose-placeholder-2026)
- [AnimatedVisibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-visibility-2026)
- [ProgressIndicator](https://zenn.dev/myougatheaxo/articles/android-compose-compose-progress-indicator-2026)
