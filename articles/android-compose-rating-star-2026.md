---
title: "★星レーティング実装ガイド — Composeでカスタム評価UIを作る"
emoji: "⭐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

Composeで**星型レーティングUI**（★★★☆☆）をカスタム実装する方法を解説します。

---

## 基本のRatingBar

```kotlin
@Composable
fun RatingBar(
    rating: Int,
    onRatingChange: (Int) -> Unit,
    maxRating: Int = 5,
    modifier: Modifier = Modifier
) {
    Row(modifier = modifier) {
        for (i in 1..maxRating) {
            IconButton(onClick = { onRatingChange(i) }) {
                Icon(
                    imageVector = if (i <= rating) Icons.Default.Star
                                  else Icons.Default.StarBorder,
                    contentDescription = "$i 星",
                    tint = if (i <= rating) Color(0xFFFFB300)
                           else Color.Gray,
                    modifier = Modifier.size(32.dp)
                )
            }
        }
    }
}

// 使用例
@Composable
fun ReviewForm() {
    var rating by remember { mutableIntStateOf(0) }

    Column(Modifier.padding(16.dp)) {
        Text("評価してください")
        RatingBar(rating = rating, onRatingChange = { rating = it })
        if (rating > 0) {
            Text("${rating}つ星の評価")
        }
    }
}
```

---

## 半星対応RatingBar

```kotlin
@Composable
fun HalfStarRatingBar(
    rating: Float,
    onRatingChange: (Float) -> Unit,
    maxRating: Int = 5
) {
    Row {
        for (i in 1..maxRating) {
            Box(
                Modifier
                    .size(36.dp)
                    .pointerInput(Unit) {
                        detectTapGestures { offset ->
                            val isHalf = offset.x < size.width / 2
                            onRatingChange(if (isHalf) i - 0.5f else i.toFloat())
                        }
                    }
            ) {
                Icon(
                    imageVector = when {
                        i <= rating -> Icons.Default.Star
                        i - 0.5f <= rating -> Icons.Default.StarHalf
                        else -> Icons.Default.StarBorder
                    },
                    contentDescription = null,
                    tint = Color(0xFFFFB300),
                    modifier = Modifier.fillMaxSize()
                )
            }
        }
    }
}
```

---

## 読み取り専用（表示のみ）

```kotlin
@Composable
fun RatingDisplay(rating: Float, reviewCount: Int) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        repeat(5) { index ->
            Icon(
                imageVector = when {
                    index < rating.toInt() -> Icons.Default.Star
                    index < rating -> Icons.Default.StarHalf
                    else -> Icons.Default.StarBorder
                },
                contentDescription = null,
                tint = Color(0xFFFFB300),
                modifier = Modifier.size(20.dp)
            )
        }
        Spacer(Modifier.width(4.dp))
        Text(
            "%.1f".format(rating),
            style = MaterialTheme.typography.bodyMedium
        )
        Text(
            " ($reviewCount件)",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.outline
        )
    }
}
```

---

## レビューカードでの使用

```kotlin
@Composable
fun ReviewCard(review: Review) {
    Card(Modifier.fillMaxWidth().padding(8.dp)) {
        Column(Modifier.padding(16.dp)) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                AsyncImage(
                    model = review.userAvatar,
                    contentDescription = null,
                    modifier = Modifier.size(40.dp).clip(CircleShape)
                )
                Spacer(Modifier.width(12.dp))
                Column {
                    Text(review.userName, style = MaterialTheme.typography.titleSmall)
                    RatingDisplay(review.rating, 0)
                }
            }
            Spacer(Modifier.height(8.dp))
            Text(review.comment, style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

---

## まとめ

- `IconButton` + `Star`/`StarBorder`で星レーティング
- `detectTapGestures`で半星対応
- `StarHalf`アイコンで0.5刻み表示
- 読み取り専用は`Icon`のみ（`IconButton`不要）
- カスタムカラーは`tint`パラメータで設定

---

8種類のAndroidアプリテンプレート（レーティングUI追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [In-App Review API](https://zenn.dev/myougatheaxo/articles/android-inapp-review-2026)
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
