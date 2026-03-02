---
title: "レーティングバー完全ガイド — 星評価/カスタムアイコン/ハーフスター"
emoji: "⭐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**レーティングバー**（星評価、ハーフスター、カスタムアイコン、タッチ操作、読み取り専用）を解説します。

---

## 基本レーティングバー

```kotlin
@Composable
fun RatingBar(
    rating: Int,
    maxRating: Int = 5,
    onRatingChange: (Int) -> Unit,
    modifier: Modifier = Modifier
) {
    Row(modifier, horizontalArrangement = Arrangement.spacedBy(4.dp)) {
        repeat(maxRating) { index ->
            val isFilled = index < rating

            Icon(
                imageVector = if (isFilled) Icons.Filled.Star else Icons.Outlined.Star,
                contentDescription = "Star ${index + 1}",
                tint = if (isFilled) Color(0xFFFFD700) else Color.Gray,
                modifier = Modifier
                    .size(32.dp)
                    .clickable { onRatingChange(index + 1) }
            )
        }
    }
}

// 使用
@Composable
fun ReviewForm() {
    var rating by remember { mutableIntStateOf(0) }

    Column(Modifier.padding(16.dp)) {
        Text("評価してください", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))
        RatingBar(rating = rating, onRatingChange = { rating = it })
        Spacer(Modifier.height(4.dp))
        Text(
            when (rating) {
                1 -> "悪い"; 2 -> "まあまあ"; 3 -> "普通"; 4 -> "良い"; 5 -> "最高！"
                else -> "タップで評価"
            },
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}
```

---

## ハーフスター対応

```kotlin
@Composable
fun HalfRatingBar(
    rating: Float,
    maxRating: Int = 5,
    onRatingChange: (Float) -> Unit,
    modifier: Modifier = Modifier
) {
    Row(modifier) {
        repeat(maxRating) { index ->
            Box(
                Modifier
                    .size(36.dp)
                    .pointerInput(Unit) {
                        detectTapGestures { offset ->
                            val half = if (offset.x < size.width / 2) 0.5f else 1f
                            onRatingChange(index + half)
                        }
                    }
            ) {
                val starFill = (rating - index).coerceIn(0f, 1f)

                // 背景（空の星）
                Icon(Icons.Outlined.Star, null, Modifier.fillMaxSize(), tint = Color.Gray)

                // 前景（塗りつぶし）
                if (starFill > 0f) {
                    Box(Modifier.fillMaxHeight().fillMaxWidth(starFill).clipToBounds()) {
                        Icon(Icons.Filled.Star, null, Modifier.size(36.dp), tint = Color(0xFFFFD700))
                    }
                }
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
            val fill = (rating - index).coerceIn(0f, 1f)
            Icon(
                imageVector = when {
                    fill >= 0.75f -> Icons.Filled.Star
                    fill >= 0.25f -> Icons.AutoMirrored.Filled.StarHalf
                    else -> Icons.Outlined.Star
                },
                contentDescription = null,
                tint = Color(0xFFFFD700),
                modifier = Modifier.size(16.dp)
            )
        }
        Spacer(Modifier.width(4.dp))
        Text("$rating", style = MaterialTheme.typography.bodySmall, fontWeight = FontWeight.Bold)
        Text("($reviewCount)", style = MaterialTheme.typography.bodySmall, color = Color.Gray)
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 整数評価 | `clickable` + `Icon` |
| ハーフスター | `detectTapGestures` + `clipToBounds` |
| 読み取り専用 | `StarHalf` アイコン |
| ラベル | `when`式で文字変換 |

- `Icons.Filled.Star` / `Outlined.Star`で塗り分け
- `detectTapGestures`で左右半分を判定してハーフスター
- 読み取り専用は`StarHalf`アイコンで0.5刻み表示
- アニメーション追加でタップフィードバック強化

---

8種類のAndroidアプリテンプレート（レビューUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-gesture-touch-2026)
- [フォーム](https://zenn.dev/myougatheaxo/articles/android-compose-form-validation-2026)
