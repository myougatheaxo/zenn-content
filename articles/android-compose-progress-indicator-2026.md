---
title: "ProgressIndicator完全ガイド — Composeでプログレスバー・インジケーターを実装"
emoji: "⏳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**CircularProgressIndicator**と**LinearProgressIndicator**の全パターンを解説します。

---

## 不確定プログレス（ローディング）

```kotlin
// 円形
CircularProgressIndicator()

// 線形
LinearProgressIndicator(modifier = Modifier.fillMaxWidth())
```

---

## 確定プログレス（進捗表示）

```kotlin
@Composable
fun DownloadProgress(progress: Float) {
    Column(Modifier.padding(16.dp)) {
        Text("${(progress * 100).toInt()}%")

        LinearProgressIndicator(
            progress = { progress },
            modifier = Modifier.fillMaxWidth().height(8.dp),
            trackColor = MaterialTheme.colorScheme.surfaceVariant
        )

        Spacer(Modifier.height(16.dp))

        CircularProgressIndicator(
            progress = { progress },
            modifier = Modifier.size(64.dp),
            strokeWidth = 6.dp
        )
    }
}
```

---

## アニメーション付きプログレス

```kotlin
@Composable
fun AnimatedProgress(targetProgress: Float) {
    val animatedProgress by animateFloatAsState(
        targetValue = targetProgress,
        animationSpec = tween(durationMillis = 1000, easing = FastOutSlowInEasing)
    )

    LinearProgressIndicator(
        progress = { animatedProgress },
        modifier = Modifier.fillMaxWidth().height(8.dp).clip(RoundedCornerShape(4.dp))
    )
}
```

---

## カスタムサーキュラープログレス

```kotlin
@Composable
fun CustomCircularProgress(
    progress: Float,
    modifier: Modifier = Modifier
) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(1000)
    )
    val textMeasurer = rememberTextMeasurer()

    Canvas(modifier = modifier.size(100.dp)) {
        // 背景
        drawArc(
            color = Color.LightGray.copy(alpha = 0.3f),
            startAngle = 0f,
            sweepAngle = 360f,
            useCenter = false,
            style = Stroke(width = 12f, cap = StrokeCap.Round)
        )
        // プログレス
        drawArc(
            color = Color(0xFF6200EE),
            startAngle = -90f,
            sweepAngle = animatedProgress * 360f,
            useCenter = false,
            style = Stroke(width = 12f, cap = StrokeCap.Round)
        )
        // テキスト
        val text = "${(animatedProgress * 100).toInt()}%"
        val textResult = textMeasurer.measure(
            text,
            TextStyle(fontSize = 20.sp, fontWeight = FontWeight.Bold)
        )
        drawText(
            textResult,
            topLeft = Offset(
                (size.width - textResult.size.width) / 2,
                (size.height - textResult.size.height) / 2
            )
        )
    }
}
```

---

## ローディングオーバーレイ

```kotlin
@Composable
fun LoadingOverlay(isLoading: Boolean, content: @Composable () -> Unit) {
    Box {
        content()

        if (isLoading) {
            Box(
                Modifier
                    .fillMaxSize()
                    .background(Color.Black.copy(alpha = 0.3f))
                    .clickable(enabled = false) {},
                contentAlignment = Alignment.Center
            ) {
                Card(shape = RoundedCornerShape(16.dp)) {
                    Column(
                        Modifier.padding(32.dp),
                        horizontalAlignment = Alignment.CenterHorizontally
                    ) {
                        CircularProgressIndicator()
                        Spacer(Modifier.height(16.dp))
                        Text("読み込み中...")
                    }
                }
            }
        }
    }
}
```

---

## ボタン内プログレス

```kotlin
@Composable
fun LoadingButton(isLoading: Boolean, onClick: () -> Unit) {
    Button(
        onClick = onClick,
        enabled = !isLoading,
        modifier = Modifier.fillMaxWidth()
    ) {
        if (isLoading) {
            CircularProgressIndicator(
                modifier = Modifier.size(20.dp),
                color = MaterialTheme.colorScheme.onPrimary,
                strokeWidth = 2.dp
            )
            Spacer(Modifier.width(8.dp))
        }
        Text(if (isLoading) "送信中..." else "送信")
    }
}
```

---

## まとめ

- `CircularProgressIndicator` / `LinearProgressIndicator`
- 不確定（引数なし）と確定（`progress`指定）
- `animateFloatAsState`でスムーズなアニメーション
- Canvas + `drawArc`でカスタムプログレス
- オーバーレイ/ボタン内で実用的なローディングUI

---

8種類のAndroidアプリテンプレート（ローディングUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Shimmer Loading効果](https://zenn.dev/myougatheaxo/articles/android-compose-shimmer-loading-2026)
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
