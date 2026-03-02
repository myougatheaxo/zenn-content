---
title: "画像加工ガイド — クロップ/フィルター/Bitmap操作"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "image"]
published: true
---

## この記事で学べること

Composeでの**画像加工**（クロップ、フィルター、Bitmap操作、Canvas描画）を解説します。

---

## 画像のクロップ表示

```kotlin
@Composable
fun CroppedImage(imageUrl: String) {
    // 中央クロップ
    AsyncImage(
        model = imageUrl,
        contentDescription = null,
        contentScale = ContentScale.Crop,
        modifier = Modifier
            .size(200.dp)
            .clip(RoundedCornerShape(16.dp))
    )
}

// 円形クロップ
@Composable
fun CircleImage(imageUrl: String) {
    AsyncImage(
        model = imageUrl,
        contentDescription = null,
        contentScale = ContentScale.Crop,
        modifier = Modifier
            .size(100.dp)
            .clip(CircleShape)
            .border(2.dp, MaterialTheme.colorScheme.primary, CircleShape)
    )
}
```

---

## カラーフィルター

```kotlin
@Composable
fun FilteredImage(imageUrl: String) {
    var selectedFilter by remember { mutableStateOf(ImageFilter.NONE) }

    Column {
        Image(
            painter = rememberAsyncImagePainter(imageUrl),
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(300.dp),
            colorFilter = when (selectedFilter) {
                ImageFilter.NONE -> null
                ImageFilter.GRAYSCALE -> ColorFilter.colorMatrix(ColorMatrix().apply { setToSaturation(0f) })
                ImageFilter.SEPIA -> ColorFilter.colorMatrix(
                    ColorMatrix(floatArrayOf(
                        0.393f, 0.769f, 0.189f, 0f, 0f,
                        0.349f, 0.686f, 0.168f, 0f, 0f,
                        0.272f, 0.534f, 0.131f, 0f, 0f,
                        0f, 0f, 0f, 1f, 0f
                    ))
                )
                ImageFilter.INVERT -> ColorFilter.colorMatrix(
                    ColorMatrix(floatArrayOf(
                        -1f, 0f, 0f, 0f, 255f,
                        0f, -1f, 0f, 0f, 255f,
                        0f, 0f, -1f, 0f, 255f,
                        0f, 0f, 0f, 1f, 0f
                    ))
                )
                ImageFilter.BRIGHTNESS -> ColorFilter.colorMatrix(
                    ColorMatrix().apply { setToScale(1.2f, 1.2f, 1.2f, 1f) }
                )
            }
        )

        LazyRow(
            contentPadding = PaddingValues(8.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(ImageFilter.entries) { filter ->
                FilterChip(
                    selected = selectedFilter == filter,
                    onClick = { selectedFilter = filter },
                    label = { Text(filter.label) }
                )
            }
        }
    }
}

enum class ImageFilter(val label: String) {
    NONE("オリジナル"),
    GRAYSCALE("モノクロ"),
    SEPIA("セピア"),
    INVERT("反転"),
    BRIGHTNESS("明るく")
}
```

---

## Bitmap操作

```kotlin
suspend fun applyBlur(context: Context, bitmap: Bitmap, radius: Float): Bitmap {
    return withContext(Dispatchers.Default) {
        val renderScript = RenderScript.create(context)
        val input = Allocation.createFromBitmap(renderScript, bitmap)
        val output = Allocation.createTyped(renderScript, input.type)
        val script = ScriptIntrinsicBlur.create(renderScript, Element.U8_4(renderScript))
        script.setRadius(radius.coerceIn(1f, 25f))
        script.setInput(input)
        script.forEach(output)
        val result = Bitmap.createBitmap(bitmap.width, bitmap.height, bitmap.config)
        output.copyTo(result)
        renderScript.destroy()
        result
    }
}

// Bitmapのリサイズ
fun resizeBitmap(bitmap: Bitmap, maxWidth: Int, maxHeight: Int): Bitmap {
    val ratio = minOf(
        maxWidth.toFloat() / bitmap.width,
        maxHeight.toFloat() / bitmap.height
    )
    val width = (bitmap.width * ratio).toInt()
    val height = (bitmap.height * ratio).toInt()
    return Bitmap.createScaledBitmap(bitmap, width, height, true)
}
```

---

## Canvas描画

```kotlin
@Composable
fun ImageWithOverlay(imageBitmap: ImageBitmap) {
    Canvas(
        modifier = Modifier
            .fillMaxWidth()
            .height(300.dp)
    ) {
        // 画像を描画
        drawImage(
            image = imageBitmap,
            dstSize = IntSize(size.width.toInt(), size.height.toInt())
        )

        // オーバーレイ（半透明黒）
        drawRect(
            color = Color.Black.copy(alpha = 0.3f),
            size = size
        )

        // テキスト用の白い帯
        drawRect(
            color = Color.White.copy(alpha = 0.8f),
            topLeft = Offset(0f, size.height - 80f),
            size = Size(size.width, 80f)
        )
    }
}
```

---

## まとめ

- `ContentScale.Crop` + `clip()`で画像クロップ
- `ColorFilter.colorMatrix()`でグレースケール/セピア/反転
- `ColorMatrix().setToSaturation(0f)`でモノクロ
- `Bitmap`操作は`Dispatchers.Default`で非同期
- `Canvas`の`drawImage()`でカスタム描画
- フィルター選択UIは`FilterChip` + `LazyRow`

---

8種類のAndroidアプリテンプレート（画像処理対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coil画像ローディング](https://zenn.dev/myougatheaxo/articles/android-compose-image-loading-2026)
- [Canvas描画ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [画像ピッカーガイド](https://zenn.dev/myougatheaxo/articles/android-compose-image-picker-2026)
