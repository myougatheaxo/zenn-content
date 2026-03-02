---
title: "カスタムModifier完全ガイド — Modifier.composed/ModifierNodeElement/描画"
emoji: "🧩"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "modifier"]
published: true
---

## この記事で学べること

**カスタムModifier**（Modifier.composed、ModifierNodeElement、描画Modifier、条件付きModifier）を解説します。

---

## 条件付きModifier

```kotlin
// 条件付きModifier拡張
fun Modifier.conditional(condition: Boolean, modifier: Modifier.() -> Modifier): Modifier {
    return if (condition) then(modifier(Modifier)) else this
}

@Composable
fun ConditionalExample() {
    var isSelected by remember { mutableStateOf(false) }

    Box(
        Modifier
            .size(100.dp)
            .conditional(isSelected) {
                border(2.dp, MaterialTheme.colorScheme.primary, RoundedCornerShape(8.dp))
            }
            .background(MaterialTheme.colorScheme.surface, RoundedCornerShape(8.dp))
            .clickable { isSelected = !isSelected }
            .padding(16.dp)
    ) {
        Text(if (isSelected) "選択中" else "未選択")
    }
}
```

---

## 描画カスタムModifier

```kotlin
fun Modifier.shimmer(): Modifier = composed {
    var size by remember { mutableStateOf(IntSize.Zero) }
    val transition = rememberInfiniteTransition(label = "shimmer")
    val startOffset by transition.animateFloat(
        initialValue = -2f,
        targetValue = 2f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000, easing = LinearEasing)
        ),
        label = "shimmer"
    )

    this
        .onSizeChanged { size = it }
        .drawWithContent {
            drawContent()
            drawRect(
                brush = Brush.linearGradient(
                    colors = listOf(
                        Color.Transparent,
                        Color.White.copy(alpha = 0.3f),
                        Color.Transparent
                    ),
                    start = Offset(size.width * startOffset, 0f),
                    end = Offset(size.width * (startOffset + 1f), size.height.toFloat())
                )
            )
        }
}

@Composable
fun ShimmerPlaceholder() {
    Box(
        Modifier
            .fillMaxWidth()
            .height(100.dp)
            .clip(RoundedCornerShape(8.dp))
            .background(Color.LightGray)
            .shimmer()
    )
}
```

---

## よく使うカスタムModifier

```kotlin
// 丸影付きクリック
fun Modifier.clickableWithRipple(onClick: () -> Unit): Modifier = composed {
    clickable(
        interactionSource = remember { MutableInteractionSource() },
        indication = ripple(bounded = true)
    ) { onClick() }
}

// デバッグ枠線
fun Modifier.debugBorder(color: Color = Color.Red): Modifier =
    border(1.dp, color)

// 最大幅制限
fun Modifier.maxContentWidth(maxWidth: Dp = 600.dp): Modifier =
    fillMaxWidth().widthIn(max = maxWidth)
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `composed` | Composable状態を使うModifier |
| `drawWithContent` | 描画カスタム |
| `onSizeChanged` | サイズ取得 |
| 拡張関数 | 再利用可能なModifier |

- `composed`でremember/animationを使うModifier
- `drawWithContent`で描画前後にカスタム描画
- 条件付きModifierで動的スタイル切り替え
- 拡張関数でプロジェクト共通のModifier定義

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Layout Modifier](https://zenn.dev/myougatheaxo/articles/android-compose-compose-layout-modifier-2026)
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [Recomposition](https://zenn.dev/myougatheaxo/articles/android-compose-compose-recomposition-2026)
