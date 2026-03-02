---
title: "drawWithCache完全ガイド — キャッシュ描画/Brush生成/パフォーマンス最適化"
emoji: "🖌️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

**drawWithCache**（キャッシュ描画、Brush/Path事前生成、描画パフォーマンス最適化）を解説します。

---

## drawWithCache基本

```kotlin
@Composable
fun DrawWithCacheBasic() {
    Box(
        Modifier
            .size(200.dp)
            .drawWithCache {
                // サイズ変更時のみ再生成
                val gradient = Brush.linearGradient(
                    listOf(Color(0xFF42A5F5), Color(0xFF7E57C2)),
                    start = Offset.Zero,
                    end = Offset(size.width, size.height)
                )
                onDrawBehind {
                    drawRoundRect(
                        brush = gradient,
                        cornerRadius = CornerRadius(16.dp.toPx())
                    )
                }
            }
    ) {
        Text(
            "キャッシュ描画",
            Modifier.align(Alignment.Center),
            color = Color.White
        )
    }
}
```

---

## Path キャッシュ

```kotlin
@Composable
fun CachedStarShape() {
    Box(
        Modifier
            .size(120.dp)
            .drawWithCache {
                val path = Path().apply {
                    val cx = size.width / 2
                    val cy = size.height / 2
                    val outerR = size.minDimension / 2
                    val innerR = outerR * 0.4f
                    for (i in 0 until 5) {
                        val outerAngle = Math.toRadians((i * 72 - 90).toDouble())
                        val innerAngle = Math.toRadians((i * 72 + 36 - 90).toDouble())
                        val ox = cx + outerR * cos(outerAngle).toFloat()
                        val oy = cy + outerR * sin(outerAngle).toFloat()
                        val ix = cx + innerR * cos(innerAngle).toFloat()
                        val iy = cy + innerR * sin(innerAngle).toFloat()
                        if (i == 0) moveTo(ox, oy) else lineTo(ox, oy)
                        lineTo(ix, iy)
                    }
                    close()
                }
                val brush = Brush.radialGradient(
                    listOf(Color(0xFFFFD54F), Color(0xFFFF6F00))
                )
                onDrawBehind {
                    drawPath(path, brush)
                }
            }
    )
}
```

---

## アニメーション連携

```kotlin
@Composable
fun AnimatedGradientBox() {
    val infiniteTransition = rememberInfiniteTransition(label = "gradient")
    val offset by infiniteTransition.animateFloat(
        initialValue = 0f, targetValue = 1f,
        animationSpec = infiniteRepeatable(
            tween(3000, easing = LinearEasing), RepeatMode.Reverse
        ), label = "offset"
    )

    Box(
        Modifier
            .fillMaxWidth()
            .height(100.dp)
            .drawWithCache {
                val brush = Brush.horizontalGradient(
                    listOf(Color(0xFF00BCD4), Color(0xFFE91E63), Color(0xFF00BCD4)),
                    startX = -size.width + size.width * 2 * offset,
                    endX = size.width * 2 * offset
                )
                onDrawBehind {
                    drawRoundRect(brush, cornerRadius = CornerRadius(12.dp.toPx()))
                }
            }
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `drawWithCache` | キャッシュ付き描画 |
| `onDrawBehind` | 背面描画コールバック |
| `onDrawWithContent` | コンテンツ+描画 |
| `Brush`キャッシュ | グラデーション再利用 |

- `drawWithCache`でBrush/Pathをサイズ変更時のみ再生成
- リコンポジション毎の再生成を回避しパフォーマンス向上
- アニメーションと組み合わせてスムーズな描画
- `onDrawBehind`/`onDrawWithContent`で描画タイミング制御

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [drawBehind/drawWithContent](https://zenn.dev/myougatheaxo/articles/android-compose-compose-draw-behind-2026)
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [グラデーション](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gradient-2026)
