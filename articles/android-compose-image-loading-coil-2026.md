---
title: "Coil画像読み込み完全ガイド — AsyncImage/キャッシュ/変換"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

**Coil**（AsyncImage、プレースホルダー、キャッシュ戦略、画像変換、プリロード）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.coil-kt.coil3:coil-compose:3.0.4")
    implementation("io.coil-kt.coil3:coil-network-okhttp:3.0.4")
}
```

---

## AsyncImage基本

```kotlin
@Composable
fun UserAvatar(imageUrl: String, modifier: Modifier = Modifier) {
    AsyncImage(
        model = imageUrl,
        contentDescription = "ユーザーアバター",
        modifier = modifier
            .size(64.dp)
            .clip(CircleShape),
        contentScale = ContentScale.Crop,
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error_image)
    )
}

// ImageRequest でカスタマイズ
@Composable
fun OptimizedImage(imageUrl: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .crossfade(true)
            .crossfade(300)
            .size(Size.ORIGINAL)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .build(),
        contentDescription = null,
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## SubcomposeAsyncImage

```kotlin
@Composable
fun DetailImage(imageUrl: String) {
    SubcomposeAsyncImage(
        model = imageUrl,
        contentDescription = "詳細画像",
        modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is AsyncImagePainter.State.Error -> {
                Box(Modifier.fillMaxSize().background(Color.LightGray),
                    contentAlignment = Alignment.Center) {
                    Icon(Icons.Default.BrokenImage, "エラー")
                }
            }
            else -> {
                SubcomposeAsyncImageContent(
                    contentScale = ContentScale.Crop
                )
            }
        }
    }
}
```

---

## ImageLoader設定

```kotlin
// Application
class MyApp : Application(), SingletonImageLoader.Factory {
    override fun newImageLoader(context: PlatformContext): ImageLoader {
        return ImageLoader.Builder(context)
            .memoryCache {
                MemoryCache.Builder()
                    .maxSizePercent(context, 0.25) // メモリの25%
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizeBytes(100 * 1024 * 1024) // 100MB
                    .build()
            }
            .crossfade(true)
            .build()
    }
}
```

---

## LazyColumn最適化

```kotlin
@Composable
fun ImageGrid(images: List<ImageItem>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(4.dp)
    ) {
        items(images, key = { it.id }) { image ->
            AsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(image.thumbnailUrl)
                    .size(300) // サムネイルサイズ指定
                    .crossfade(true)
                    .build(),
                contentDescription = image.title,
                modifier = Modifier
                    .aspectRatio(1f)
                    .padding(2.dp)
                    .clip(RoundedCornerShape(4.dp)),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

---

## プリロード

```kotlin
@Composable
fun PreloadImages(urls: List<String>) {
    val context = LocalContext.current
    val imageLoader = context.imageLoader

    LaunchedEffect(urls) {
        urls.forEach { url ->
            val request = ImageRequest.Builder(context)
                .data(url)
                .size(Size.ORIGINAL)
                .build()
            imageLoader.enqueue(request)
        }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `AsyncImage` | 基本画像読み込み |
| `SubcomposeAsyncImage` | 状態別UI |
| `ImageRequest` | 詳細設定 |
| `ImageLoader` | グローバル設定 |
| `MemoryCache`/`DiskCache` | キャッシュ管理 |

- `crossfade(true)`でフェードインアニメーション
- `size()`でリサイズしてメモリ節約
- `placeholder`/`error`でUX向上
- `LazyColumn`ではサムネイルサイズを指定

---

8種類のAndroidアプリテンプレート（Coil画像読み込み対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-tips-2026)
- [パフォーマンスプロファイリング](https://zenn.dev/myougatheaxo/articles/android-compose-performance-profiling-2026)
- [Paging3](https://zenn.dev/myougatheaxo/articles/android-compose-paging3-compose-2026)
