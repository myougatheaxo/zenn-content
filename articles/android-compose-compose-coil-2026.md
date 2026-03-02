---
title: "Compose Coil完全ガイド — AsyncImage/キャッシュ/プレースホルダー/変換"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

**Compose Coil**（AsyncImage、キャッシュ制御、プレースホルダー、画像変換）を解説します。

---

## 基本AsyncImage

```kotlin
// build.gradle
// implementation "io.coil-kt:coil-compose:2.6.0"

@Composable
fun BasicImageDemo() {
    AsyncImage(
        model = "https://example.com/image.jpg",
        contentDescription = "サンプル画像",
        modifier = Modifier.fillMaxWidth().height(200.dp),
        contentScale = ContentScale.Crop
    )
}

// プレースホルダー付き
@Composable
fun ImageWithPlaceholder(url: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .build(),
        contentDescription = null,
        modifier = Modifier.size(100.dp).clip(CircleShape),
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error),
        contentScale = ContentScale.Crop
    )
}
```

---

## SubcomposeAsyncImage

```kotlin
@Composable
fun SmartImage(url: String, modifier: Modifier = Modifier) {
    SubcomposeAsyncImage(
        model = url,
        contentDescription = null,
        modifier = modifier
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator(Modifier.size(24.dp))
                }
            }
            is AsyncImagePainter.State.Error -> {
                Box(Modifier.fillMaxSize().background(Color.Gray), contentAlignment = Alignment.Center) {
                    Icon(Icons.Default.BrokenImage, "エラー", tint = Color.White)
                }
            }
            else -> {
                SubcomposeAsyncImageContent(contentScale = ContentScale.Crop)
            }
        }
    }
}
```

---

## ImageLoader設定

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ImageModule {
    @Provides
    @Singleton
    fun provideImageLoader(@ApplicationContext context: Context): ImageLoader =
        ImageLoader.Builder(context)
            .memoryCache {
                MemoryCache.Builder(context)
                    .maxSizePercent(0.25)
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizePercent(0.02)
                    .build()
            }
            .crossfade(true)
            .respectCacheHeaders(false)
            .build()
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AsyncImage` | 基本画像読み込み |
| `SubcomposeAsyncImage` | カスタムローディング |
| `ImageRequest` | 詳細設定 |
| `ImageLoader` | グローバル設定 |

- `AsyncImage`でURL画像を簡単に表示
- `crossfade(true)`でフェードインアニメーション
- `SubcomposeAsyncImage`で読み込み状態別にUI切替
- メモリ/ディスクキャッシュでパフォーマンス最適化

---

8種類のAndroidアプリテンプレート（画像読み込み対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Glide](https://zenn.dev/myougatheaxo/articles/android-compose-compose-glide-2026)
- [Compose ImageCropper](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-cropper-2026)
- [Compose Placeholder](https://zenn.dev/myougatheaxo/articles/android-compose-compose-placeholder-2026)
