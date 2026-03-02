---
title: "ProgressIndicator完全ガイド — Circular/Linear/カスタム/Skeleton"
emoji: "⏳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**ProgressIndicator**（CircularProgressIndicator、LinearProgressIndicator、カスタムプログレス、Skeletonローディング）を解説します。

---

## 基本ProgressIndicator

```kotlin
@Composable
fun ProgressExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // 不確定（回転）
        CircularProgressIndicator()

        // 確定（進捗付き）
        var progress by remember { mutableFloatStateOf(0.3f) }
        CircularProgressIndicator(progress = { progress })

        // 線形（不確定）
        LinearProgressIndicator(Modifier.fillMaxWidth())

        // 線形（確定）
        LinearProgressIndicator(
            progress = { progress },
            modifier = Modifier.fillMaxWidth()
        )

        Slider(value = progress, onValueChange = { progress = it })
    }
}
```

---

## カスタムCircularProgress

```kotlin
@Composable
fun CustomCircularProgress(progress: Float, modifier: Modifier = Modifier) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(1000),
        label = "progress"
    )

    Box(modifier.size(120.dp), contentAlignment = Alignment.Center) {
        Canvas(Modifier.fillMaxSize()) {
            // 背景リング
            drawArc(
                color = Color.LightGray,
                startAngle = -90f,
                sweepAngle = 360f,
                useCenter = false,
                style = Stroke(12.dp.toPx(), cap = StrokeCap.Round)
            )
            // 進捗リング
            drawArc(
                color = Color(0xFF6200EA),
                startAngle = -90f,
                sweepAngle = 360f * animatedProgress,
                useCenter = false,
                style = Stroke(12.dp.toPx(), cap = StrokeCap.Round)
            )
        }
        Text(
            "${(animatedProgress * 100).toInt()}%",
            style = MaterialTheme.typography.headlineSmall,
            fontWeight = FontWeight.Bold
        )
    }
}
```

---

## Skeletonローディング

```kotlin
@Composable
fun SkeletonLoading() {
    val shimmer = rememberInfiniteTransition(label = "shimmer")
    val alpha by shimmer.animateFloat(
        initialValue = 0.3f,
        targetValue = 0.9f,
        animationSpec = infiniteRepeatable(
            animation = tween(800),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Column(Modifier.padding(16.dp)) {
        repeat(3) {
            Row(Modifier.padding(vertical = 8.dp)) {
                Box(
                    Modifier
                        .size(48.dp)
                        .clip(CircleShape)
                        .background(Color.Gray.copy(alpha = alpha))
                )
                Spacer(Modifier.width(12.dp))
                Column {
                    Box(
                        Modifier
                            .fillMaxWidth(0.7f)
                            .height(16.dp)
                            .clip(RoundedCornerShape(4.dp))
                            .background(Color.Gray.copy(alpha = alpha))
                    )
                    Spacer(Modifier.height(8.dp))
                    Box(
                        Modifier
                            .fillMaxWidth(0.4f)
                            .height(12.dp)
                            .clip(RoundedCornerShape(4.dp))
                            .background(Color.Gray.copy(alpha = alpha))
                    )
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `CircularProgressIndicator` | 円形プログレス |
| `LinearProgressIndicator` | 線形プログレス |
| `Canvas` + `drawArc` | カスタムリング |
| `infiniteTransition` | Skeletonシマー |

- 不確定プログレスはパラメータなし
- 確定プログレスは`progress`パラメータ
- `Canvas`+`drawArc`でカスタムリング
- Skeleton UIで読み込み中のプレースホルダー

---

8種類のAndroidアプリテンプレート（ローディングUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [PullToRefresh](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pull-refresh-2026)
- [AnimatedVisibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-visibility-2026)
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
