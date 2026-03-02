---
title: "BlendMode完全ガイド — 描画合成/マスク/エフェクト"
emoji: "🎭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "graphics"]
published: true
---

## この記事で学べること

**BlendMode**（描画合成モード、マスクエフェクト、レイヤー合成）を解説します。

---

## BlendMode基本

```kotlin
@Composable
fun BlendModeBasic() {
    Canvas(Modifier.size(200.dp)) {
        // 背景円（赤）
        drawCircle(
            color = Color.Red,
            radius = 80.dp.toPx(),
            center = Offset(size.width * 0.4f, size.height * 0.4f)
        )
        // 前景円（青）をMultiplyで合成
        drawCircle(
            color = Color.Blue,
            radius = 80.dp.toPx(),
            center = Offset(size.width * 0.6f, size.height * 0.6f),
            blendMode = BlendMode.Multiply
        )
    }
}
```

---

## テキストマスク

```kotlin
@Composable
fun TextMaskEffect() {
    val textMeasurer = rememberTextMeasurer()

    Canvas(Modifier.fillMaxWidth().height(100.dp)) {
        // 背景グラデーション描画
        drawRect(
            brush = Brush.horizontalGradient(
                listOf(Color(0xFFFF6B6B), Color(0xFF4ECDC4), Color(0xFF45B7D1))
            )
        )
        // テキストでくり抜き
        val textResult = textMeasurer.measure(
            AnnotatedString("COMPOSE"),
            style = TextStyle(fontSize = 60.sp, fontWeight = FontWeight.Black)
        )
        drawText(
            textResult,
            color = Color.White,
            topLeft = Offset(
                (size.width - textResult.size.width) / 2,
                (size.height - textResult.size.height) / 2
            ),
            blendMode = BlendMode.DstIn
        )
    }
}
```

---

## レイヤー合成

```kotlin
@Composable
fun LayerBlendDemo() {
    Canvas(Modifier.size(200.dp)) {
        // レイヤー内で合成
        withTransform({
            // 変換なし（グループ化のため）
        }) {
            drawRect(
                brush = Brush.linearGradient(
                    listOf(Color.Yellow, Color.Red)
                ),
                size = Size(size.width * 0.7f, size.height * 0.7f)
            )
            drawCircle(
                color = Color.Cyan,
                radius = size.minDimension * 0.3f,
                center = Offset(size.width * 0.6f, size.height * 0.6f),
                blendMode = BlendMode.Screen
            )
        }
    }
}
```

---

## まとめ

| BlendMode | 効果 |
|-----------|------|
| `Multiply` | 暗い色が残る |
| `Screen` | 明るい色が残る |
| `Overlay` | コントラスト強調 |
| `DstIn` | マスク（くり抜き） |

- `BlendMode`でCanvas描画の合成方法を制御
- `DstIn`/`SrcIn`でマスクエフェクト
- `Multiply`/`Screen`で色の合成
- レイヤーグループで複雑な合成も可能

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Brush](https://zenn.dev/myougatheaxo/articles/android-compose-compose-brush-2026)
- [Path描画](https://zenn.dev/myougatheaxo/articles/android-compose-compose-path-2026)
- [drawWithCache](https://zenn.dev/myougatheaxo/articles/android-compose-compose-draw-with-cache-2026)
