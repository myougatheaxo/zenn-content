---
title: "カラーピッカー完全ガイド — Canvas描画/HSV変換/カスタムピッカー"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**カラーピッカー**（Canvas描画、HSVカラーモデル、カスタムカラーピッカー）を解説します。

---

## HSVカラーピッカー

```kotlin
@Composable
fun ColorPickerScreen() {
    var selectedColor by remember { mutableStateOf(Color.Red) }
    var hue by remember { mutableFloatStateOf(0f) }
    var saturation by remember { mutableFloatStateOf(1f) }
    var brightness by remember { mutableFloatStateOf(1f) }

    Column(Modifier.padding(16.dp)) {
        // プレビュー
        Box(
            Modifier
                .fillMaxWidth()
                .height(80.dp)
                .background(selectedColor, RoundedCornerShape(12.dp))
        )

        Spacer(Modifier.height(16.dp))

        // Hueスライダー
        Text("色相: ${hue.toInt()}°")
        Slider(
            value = hue,
            onValueChange = {
                hue = it
                selectedColor = Color.hsv(hue, saturation, brightness)
            },
            valueRange = 0f..360f
        )

        // Saturationスライダー
        Text("彩度: ${(saturation * 100).toInt()}%")
        Slider(
            value = saturation,
            onValueChange = {
                saturation = it
                selectedColor = Color.hsv(hue, saturation, brightness)
            }
        )

        // Brightnessスライダー
        Text("明度: ${(brightness * 100).toInt()}%")
        Slider(
            value = brightness,
            onValueChange = {
                brightness = it
                selectedColor = Color.hsv(hue, saturation, brightness)
            }
        )

        // HEX表示
        val hex = String.format("#%06X", 0xFFFFFF and selectedColor.toArgb())
        Text("HEX: $hex", style = MaterialTheme.typography.titleMedium)
    }
}
```

---

## プリセットカラーグリッド

```kotlin
@Composable
fun PresetColorPicker(
    selectedColor: Color,
    onColorSelected: (Color) -> Unit
) {
    val presets = listOf(
        Color.Red, Color(0xFFFF5722), Color(0xFFFF9800),
        Color(0xFFFFC107), Color(0xFF4CAF50), Color(0xFF009688),
        Color(0xFF2196F3), Color(0xFF3F51B5), Color(0xFF9C27B0),
        Color(0xFF795548), Color(0xFF607D8B), Color.Black
    )

    LazyVerticalGrid(
        columns = GridCells.Fixed(6),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(presets) { color ->
            Box(
                Modifier
                    .size(48.dp)
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

## まとめ

| 機能 | 実装方法 |
|------|---------|
| HSVスライダー | `Color.hsv()` + `Slider` |
| プリセット | `LazyVerticalGrid` + `CircleShape` |
| HEX表示 | `String.format("#%06X")` |
| Canvas描画 | `drawRect` + `Brush.horizontalGradient` |

- `Color.hsv()`でHSVモデルから色変換
- `Slider`で色相/彩度/明度を個別調整
- プリセットカラーグリッドで簡単選択
- `toArgb()`でHEX文字列に変換

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [ダイナミックテーマ](https://zenn.dev/myougatheaxo/articles/android-compose-dynamic-theme-2026)
- [Slider](https://zenn.dev/myougatheaxo/articles/android-compose-compose-slider-2026)
