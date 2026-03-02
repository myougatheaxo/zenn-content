---
title: "画像読み込みガイド — CoilでCompose画像表示を最適化"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

**Coil**を使ったCompose画像読み込みの実装方法（キャッシュ・プレースホルダー・変換）を解説します。

---

## 依存関係

```kotlin
dependencies {
    implementation("io.coil-kt:coil-compose:2.7.0")
}
```

---

## 基本の画像表示

```kotlin
@Composable
fun BasicImage() {
    AsyncImage(
        model = "https://example.com/photo.jpg",
        contentDescription = "写真",
        modifier = Modifier.size(200.dp),
        contentScale = ContentScale.Crop
    )
}
```

---

## プレースホルダー・エラー表示

```kotlin
@Composable
fun ImageWithPlaceholder(url: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .build(),
        contentDescription = null,
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error_image),
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .clip(RoundedCornerShape(12.dp)),
        contentScale = ContentScale.Crop
    )
}
```

---

## SubcomposeAsyncImage

```kotlin
@Composable
fun CustomLoadingImage(url: String) {
    SubcomposeAsyncImage(
        model = url,
        contentDescription = null,
        modifier = Modifier.fillMaxWidth().height(200.dp)
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(
                    Modifier.fillMaxSize().background(Color.LightGray),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator(Modifier.size(32.dp))
                }
            }
            is AsyncImagePainter.State.Error -> {
                Box(
                    Modifier.fillMaxSize().background(Color.LightGray),
                    contentAlignment = Alignment.Center
                ) {
                    Icon(Icons.Default.BrokenImage, "エラー", Modifier.size(48.dp))
                }
            }
            else -> {
                SubcomposeAsyncImageContent(
                    modifier = Modifier.clip(RoundedCornerShape(12.dp)),
                    contentScale = ContentScale.Crop
                )
            }
        }
    }
}
```

---

## 円形画像（アバター）

```kotlin
@Composable
fun AvatarImage(url: String, size: Dp = 60.dp) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .transformations(CircleCropTransformation())
            .build(),
        contentDescription = "アバター",
        modifier = Modifier
            .size(size)
            .clip(CircleShape)
            .border(2.dp, MaterialTheme.colorScheme.primary, CircleShape),
        contentScale = ContentScale.Crop
    )
}
```

---

## 画像グリッド

```kotlin
@Composable
fun PhotoGrid(photos: List<String>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        contentPadding = PaddingValues(4.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp),
        horizontalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        items(photos) { url ->
            AsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(url)
                    .crossfade(true)
                    .size(300)
                    .build(),
                contentDescription = null,
                modifier = Modifier
                    .aspectRatio(1f)
                    .clip(RoundedCornerShape(4.dp)),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

---

## ImageLoaderのカスタマイズ

```kotlin
class MyApp : Application(), ImageLoaderFactory {
    override fun newImageLoader(): ImageLoader {
        return ImageLoader.Builder(this)
            .crossfade(true)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .diskCache {
                DiskCache.Builder()
                    .directory(cacheDir.resolve("image_cache"))
                    .maxSizePercent(0.02)
                    .build()
            }
            .memoryCache {
                MemoryCache.Builder(this)
                    .maxSizePercent(0.25)
                    .build()
            }
            .build()
    }
}
```

---

## まとめ

- `AsyncImage`で基本的なURL画像読み込み
- `ImageRequest.Builder`でcrossfade・変換設定
- `SubcomposeAsyncImage`でカスタムローディングUI
- `CircleCropTransformation()`で円形クリップ
- `.size()`でサムネイル最適化
- `ImageLoaderFactory`でキャッシュポリシー設定

---

8種類のAndroidアプリテンプレート（画像表示最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyVerticalGridガイド](https://zenn.dev/myougatheaxo/articles/compose-lazy-grid-2026)
- [CameraX実装ガイド](https://zenn.dev/myougatheaxo/articles/android-camerax-compose-2026)
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
