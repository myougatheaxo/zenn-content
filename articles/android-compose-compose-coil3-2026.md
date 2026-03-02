---
title: "Compose Coil3完全ガイド — AsyncImage/プレースホルダー/キャッシュ/トランスフォーム"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

**Compose Coil3**（AsyncImage、プレースホルダー、ディスクキャッシュ、画像変換、KMP対応）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation("io.coil-kt.coil3:coil-compose:3.0.4")
    implementation("io.coil-kt.coil3:coil-network-okhttp:3.0.4")
}
```

---

## AsyncImage

```kotlin
@Composable
fun ImageDemo() {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data("https://example.com/photo.jpg")
            .crossfade(true)
            .build(),
        contentDescription = "写真",
        modifier = Modifier.fillMaxWidth().height(200.dp).clip(RoundedCornerShape(16.dp)),
        contentScale = ContentScale.Crop,
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error_image)
    )
}

// SubcomposeAsyncImage（ローディング状態カスタマイズ）
@Composable
fun CustomLoadingImage(url: String) {
    SubcomposeAsyncImage(
        model = url,
        contentDescription = null,
        modifier = Modifier.size(200.dp)
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is AsyncImagePainter.State.Error -> {
                Icon(Icons.Default.BrokenImage, contentDescription = "エラー",
                    modifier = Modifier.size(48.dp))
            }
            else -> {
                SubcomposeAsyncImageContent(contentScale = ContentScale.Crop)
            }
        }
    }
}
```

---

## カスタムImageLoader

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ImageModule {
    @Provides
    @Singleton
    fun provideImageLoader(@ApplicationContext context: Context): ImageLoader =
        ImageLoader.Builder(context)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizeBytes(100L * 1024 * 1024) // 100MB
                    .build()
            }
            .memoryCache {
                MemoryCache.Builder()
                    .maxSizePercent(context, 0.25) // 25%のメモリ
                    .build()
            }
            .respectCacheHeaders(false)
            .build()
}

// アプリ全体に設定
@Composable
fun App() {
    val imageLoader = hiltViewModel<AppViewModel>().imageLoader
    setSingletonImageLoaderFactory { imageLoader }

    MaterialTheme {
        MainScreen()
    }
}
```

---

## 画像リスト

```kotlin
@Composable
fun ImageGrid(urls: List<String>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        contentPadding = PaddingValues(4.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp),
        horizontalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        items(urls) { url ->
            AsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(url)
                    .crossfade(true)
                    .size(Size.ORIGINAL)
                    .build(),
                contentDescription = null,
                modifier = Modifier.aspectRatio(1f).clip(RoundedCornerShape(8.dp)),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AsyncImage` | 画像ロード |
| `ImageRequest` | リクエスト設定 |
| `ImageLoader` | ローダー設定 |
| `DiskCache` | ディスクキャッシュ |

- Coil3はKMP対応の最新画像ローダー
- `AsyncImage`でCompose向けの画像表示
- `crossfade(true)`でフェードインアニメーション
- メモリ+ディスクの2層キャッシュで高速表示

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ImageLoading](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
- [Compose GraphicsLayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-graphicsLayer-2026)
