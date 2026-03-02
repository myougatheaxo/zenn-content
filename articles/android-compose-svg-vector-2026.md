---
title: "SVG/VectorDrawable完全ガイド — アイコン/アニメーション/動的カラー"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "design"]
published: true
---

## この記事で学べること

**SVG/VectorDrawable**（Vector Asset、ImageVector、アニメーション、動的カラー変更）を解説します。

---

## ImageVector基本

```kotlin
@Composable
fun VectorIcon() {
    // Material Icons
    Icon(
        imageVector = Icons.Default.Favorite,
        contentDescription = "お気に入り",
        tint = MaterialTheme.colorScheme.primary,
        modifier = Modifier.size(24.dp)
    )

    // リソースから
    Icon(
        painter = painterResource(R.drawable.ic_custom_icon),
        contentDescription = "カスタム",
        tint = Color.Unspecified // 元の色を維持
    )
}
```

---

## カスタムImageVector

```kotlin
val StarIcon: ImageVector = ImageVector.Builder(
    name = "Star",
    defaultWidth = 24.dp,
    defaultHeight = 24.dp,
    viewportWidth = 24f,
    viewportHeight = 24f
).apply {
    path(
        fill = SolidColor(Color.Black),
        pathData = PathData {
            moveTo(12f, 2f)
            lineTo(15.09f, 8.26f)
            lineTo(22f, 9.27f)
            lineTo(17f, 14.14f)
            lineTo(18.18f, 21.02f)
            lineTo(12f, 17.77f)
            lineTo(5.82f, 21.02f)
            lineTo(7f, 14.14f)
            lineTo(2f, 9.27f)
            lineTo(8.91f, 8.26f)
            close()
        }
    )
}.build()

@Composable
fun RatingStars(rating: Int, maxRating: Int = 5) {
    Row {
        repeat(maxRating) { index ->
            Icon(
                imageVector = StarIcon,
                contentDescription = null,
                tint = if (index < rating) Color(0xFFFFD700) else Color.Gray,
                modifier = Modifier.size(20.dp)
            )
        }
    }
}
```

---

## AnimatedVectorDrawable

```kotlin
@Composable
fun AnimatedCheckmark() {
    var atEnd by remember { mutableStateOf(false) }

    val painter = rememberAnimatedVectorPainter(
        animatedImageVector = AnimatedImageVector.animatedVectorResource(
            R.drawable.avd_checkmark
        ),
        atEnd = atEnd
    )

    Image(
        painter = painter,
        contentDescription = "チェックマーク",
        modifier = Modifier
            .size(64.dp)
            .clickable { atEnd = !atEnd }
    )
}
```

---

## 動的カラー変更

```kotlin
@Composable
fun DynamicColorIcon(
    icon: ImageVector,
    isSelected: Boolean,
    modifier: Modifier = Modifier
) {
    val color by animateColorAsState(
        targetValue = if (isSelected)
            MaterialTheme.colorScheme.primary
        else
            MaterialTheme.colorScheme.onSurfaceVariant,
        animationSpec = tween(300),
        label = "iconColor"
    )

    val scale by animateFloatAsState(
        targetValue = if (isSelected) 1.2f else 1f,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
        label = "iconScale"
    )

    Icon(
        imageVector = icon,
        contentDescription = null,
        tint = color,
        modifier = modifier
            .size(24.dp)
            .graphicsLayer { scaleX = scale; scaleY = scale }
    )
}
```

---

## まとめ

| 方法 | 用途 |
|------|------|
| `Icons.Default.*` | Material標準アイコン |
| `painterResource` | XML VectorDrawable |
| `ImageVector.Builder` | コードで定義 |
| `AnimatedImageVector` | アニメーション |

- `ImageVector`でスケーラブルなアイコン
- `AnimatedImageVector`でアニメーションアイコン
- `tint`で動的にカラー変更
- コードで定義すればXMLリソース不要

---

8種類のAndroidアプリテンプレート（カスタムアイコン対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-custom-drawing-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
