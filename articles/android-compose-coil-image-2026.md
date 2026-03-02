---
title: "Coil画像読み込みガイド — AsyncImage/キャッシュ/プレースホルダー"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

**Coil**でのCompose画像読み込み（AsyncImage、キャッシュ、変換、エラー処理）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("io.coil-kt:coil-compose:2.7.0")
}
```

---

## AsyncImage基本

```kotlin
@Composable
fun BasicImage(url: String) {
    AsyncImage(
        model = url,
        contentDescription = "画像",
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .clip(RoundedCornerShape(12.dp)),
        contentScale = ContentScale.Crop
    )
}
```

---

## プレースホルダー/エラー画像

```kotlin
@Composable
fun ImageWithPlaceholder(url: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .build(),
        contentDescription = null,
        modifier = Modifier
            .size(120.dp)
            .clip(CircleShape),
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error_image),
        contentScale = ContentScale.Crop
    )
}
```

---

## SubcomposeAsyncImage

```kotlin
@Composable
fun DetailedImage(url: String) {
    SubcomposeAsyncImage(
        model = url,
        contentDescription = null,
        modifier = Modifier.fillMaxWidth()
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(
                    Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
            is AsyncImagePainter.State.Error -> {
                Box(
                    Modifier.fillMaxSize().background(Color.LightGray),
                    contentAlignment = Alignment.Center
                ) {
                    Icon(Icons.Default.BrokenImage, null)
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

## キャッシュ設定

```kotlin
// Application.onCreate()
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        val imageLoader = ImageLoader.Builder(this)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .memoryCache {
                MemoryCache.Builder(this)
                    .maxSizePercent(0.25) // メモリの25%
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(cacheDir.resolve("image_cache"))
                    .maxSizeBytes(50L * 1024 * 1024) // 50MB
                    .build()
            }
            .build()

        Coil.setImageLoader(imageLoader)
    }
}
```

---

## まとめ

- `AsyncImage`でURLから簡単に画像表示
- `ImageRequest.Builder`でcrossfade/変換設定
- `placeholder`/`error`でローディング/エラー画像
- `SubcomposeAsyncImage`で状態別のカスタムUI
- `MemoryCache`/`DiskCache`でキャッシュ管理
- `ContentScale.Crop`/`Fit`で表示モード指定

---

8種類のAndroidアプリテンプレート（画像表示最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [画像加工/フィルター](https://zenn.dev/myougatheaxo/articles/android-compose-image-crop-filter-2026)
- [LazyColumnパフォーマンス](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
- [GridLayoutガイド](https://zenn.dev/myougatheaxo/articles/android-compose-grid-layout-2026)
