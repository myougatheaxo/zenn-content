---
title: "カラーピッカー実装ガイド — ComposeでCanvasベースの色選択UI"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

Composeで**カラーピッカー**（色相バー + 彩度/明度パネル）を自作する方法を解説します。

---

## プリセットカラーピッカー

```kotlin
@Composable
fun PresetColorPicker(
    selectedColor: Color,
    onColorSelected: (Color) -> Unit
) {
    val colors = listOf(
        Color.Red, Color(0xFFFF9800), Color(0xFFFFEB3B),
        Color(0xFF4CAF50), Color(0xFF2196F3), Color(0xFF9C27B0),
        Color(0xFF795548), Color.Black, Color.Gray
    )

    FlowRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        colors.forEach { color ->
            Box(
                modifier = Modifier
                    .size(40.dp)
                    .clip(CircleShape)
                    .background(color)
                    .border(
                        width = if (color == selectedColor) 3.dp else 0.dp,
                        color = MaterialTheme.colorScheme.primary,
                        shape = CircleShape
                    )
                    .clickable { onColorSelected(color) }
            )
        }
    }
}
```

---

## 色相バー（HueBar）

```kotlin
@Composable
fun HueBar(
    hue: Float,
    onHueChange: (Float) -> Unit,
    modifier: Modifier = Modifier
) {
    Canvas(
        modifier = modifier
            .fillMaxWidth()
            .height(30.dp)
            .clip(RoundedCornerShape(15.dp))
            .pointerInput(Unit) {
                detectDragGestures { change, _ ->
                    change.consume()
                    val newHue = (change.position.x / size.width).coerceIn(0f, 1f) * 360f
                    onHueChange(newHue)
                }
            }
            .pointerInput(Unit) {
                detectTapGestures { offset ->
                    val newHue = (offset.x / size.width).coerceIn(0f, 1f) * 360f
                    onHueChange(newHue)
                }
            }
    ) {
        val hueBrush = Brush.horizontalGradient(
            colors = (0..360 step 30).map { Color.hsv(it.toFloat(), 1f, 1f) }
        )
        drawRect(brush = hueBrush)

        // インジケーター
        val x = (hue / 360f) * size.width
        drawCircle(
            Color.White,
            radius = 14f,
            center = Offset(x, size.height / 2),
            style = Stroke(width = 3f)
        )
    }
}
```

---

## カラーピッカー全体

```kotlin
@Composable
fun ColorPicker(
    initialColor: Color = Color.Red,
    onColorSelected: (Color) -> Unit
) {
    var hue by remember { mutableFloatStateOf(0f) }
    var saturation by remember { mutableFloatStateOf(1f) }
    var value by remember { mutableFloatStateOf(1f) }

    val currentColor = Color.hsv(hue, saturation, value)

    Column(Modifier.padding(16.dp)) {
        // プレビュー
        Box(
            Modifier
                .fillMaxWidth()
                .height(60.dp)
                .clip(RoundedCornerShape(12.dp))
                .background(currentColor)
        )

        Spacer(Modifier.height(16.dp))

        // 色相
        Text("色相", style = MaterialTheme.typography.labelMedium)
        HueBar(hue = hue, onHueChange = { hue = it })

        Spacer(Modifier.height(12.dp))

        // 彩度
        Text("彩度", style = MaterialTheme.typography.labelMedium)
        Slider(
            value = saturation,
            onValueChange = { saturation = it },
            colors = SliderDefaults.colors(
                thumbColor = currentColor,
                activeTrackColor = currentColor
            )
        )

        // 明度
        Text("明度", style = MaterialTheme.typography.labelMedium)
        Slider(
            value = value,
            onValueChange = { value = it },
            colors = SliderDefaults.colors(
                thumbColor = currentColor,
                activeTrackColor = currentColor
            )
        )

        Spacer(Modifier.height(16.dp))

        Button(
            onClick = { onColorSelected(currentColor) },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("この色を選択")
        }
    }
}
```

---

## まとめ

- プリセット: `FlowRow` + 色付き`Box`で簡単実装
- 色相バー: `Canvas` + `horizontalGradient` + `detectDragGestures`
- 彩度/明度: `Slider`で0-1の範囲調整
- `Color.hsv()`でHSV→Color変換
- BottomSheetに配置して使うのがおすすめ

---

8種類のAndroidアプリテンプレート（カスタムUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [カスタムテーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-custom-2026)
- [ジェスチャー入門](https://zenn.dev/myougatheaxo/articles/compose-gesture-touch-2026)
