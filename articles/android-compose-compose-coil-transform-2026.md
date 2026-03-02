---
title: "Coil Transform完全ガイド — 丸型/ぼかし/グレースケール/カスタム変換"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

**Coil Transform**（CircleCropTransformation、BlurTransformation、カスタム変換）を解説します。

---

## 基本Transform

```kotlin
@Composable
fun TransformExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 丸型クロップ
        AsyncImage(
            model = ImageRequest.Builder(LocalContext.current)
                .data("https://example.com/avatar.jpg")
                .transformations(CircleCropTransformation())
                .build(),
            contentDescription = "アバター",
            modifier = Modifier.size(100.dp)
        )

        // 角丸
        AsyncImage(
            model = ImageRequest.Builder(LocalContext.current)
                .data("https://example.com/photo.jpg")
                .transformations(RoundedCornersTransformation(24f))
                .build(),
            contentDescription = null,
            modifier = Modifier.fillMaxWidth().height(200.dp)
        )

        // 複数変換
        AsyncImage(
            model = ImageRequest.Builder(LocalContext.current)
                .data("https://example.com/photo.jpg")
                .transformations(
                    CircleCropTransformation(),
                    GrayscaleTransformation()
                )
                .build(),
            contentDescription = null,
            modifier = Modifier.size(100.dp)
        )
    }
}
```

---

## カスタムTransformation

```kotlin
class TintTransformation(private val color: Int) : Transformation {
    override val cacheKey = "tint_$color"

    override suspend fun transform(input: Bitmap, size: Size): Bitmap {
        val output = input.copy(Bitmap.Config.ARGB_8888, true)
        val canvas = android.graphics.Canvas(output)
        val paint = android.graphics.Paint().apply {
            colorFilter = PorterDuffColorFilter(color, PorterDuff.Mode.SRC_ATOP)
        }
        canvas.drawBitmap(output, 0f, 0f, paint)
        return output
    }
}

@Composable
fun TintedImage() {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data("https://example.com/icon.png")
            .transformations(TintTransformation(android.graphics.Color.RED))
            .build(),
        contentDescription = null,
        modifier = Modifier.size(64.dp)
    )
}
```

---

## プレースホルダー連携

```kotlin
@Composable
fun TransformedWithPlaceholder(url: String) {
    SubcomposeAsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .transformations(RoundedCornersTransformation(16f))
            .build(),
        contentDescription = null,
        modifier = Modifier.fillMaxWidth().height(200.dp)
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(
                    Modifier
                        .fillMaxSize()
                        .clip(RoundedCornerShape(16.dp))
                        .background(Color.LightGray)
                )
            }
            else -> SubcomposeAsyncImageContent()
        }
    }
}
```

---

## まとめ

| Transform | 用途 |
|-----------|------|
| `CircleCropTransformation` | 丸型クロップ |
| `RoundedCornersTransformation` | 角丸 |
| `GrayscaleTransformation` | グレースケール |
| カスタム`Transformation` | 独自変換 |

- `transformations()`で画像読み込み時に変換適用
- 複数のTransformationを組み合わせ可能
- `cacheKey`でキャッシュの正確な一致を保証
- `crossfade(true)`でフェードインアニメーション

---

8種類のAndroidアプリテンプレート（画像対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camerax-analysis-2026)
- [Clip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clip-2026)
