---
title: "画像読み込み完全ガイド — Coil3/AsyncImage/キャッシュ/プレースホルダー"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

**画像読み込み**（Coil3、AsyncImage、メモリ/ディスクキャッシュ、プレースホルダー）を解説します。

---

## AsyncImage基本

```kotlin
@Composable
fun ImageExample() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 基本
        AsyncImage(
            model = "https://example.com/photo.jpg",
            contentDescription = "写真",
            modifier = Modifier.fillMaxWidth().height(200.dp),
            contentScale = ContentScale.Crop
        )

        // プレースホルダー/エラー
        AsyncImage(
            model = ImageRequest.Builder(LocalContext.current)
                .data("https://example.com/photo.jpg")
                .crossfade(true)
                .build(),
            contentDescription = null,
            placeholder = painterResource(R.drawable.placeholder),
            error = painterResource(R.drawable.error),
            modifier = Modifier.size(120.dp).clip(CircleShape),
            contentScale = ContentScale.Crop
        )
    }
}
```

---

## SubcomposeAsyncImage

```kotlin
@Composable
fun DetailedImageLoading(url: String) {
    SubcomposeAsyncImage(
        model = url,
        contentDescription = null,
        modifier = Modifier.fillMaxWidth().height(250.dp)
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is AsyncImagePainter.State.Error -> {
                Box(Modifier.fillMaxSize().background(Color.LightGray), contentAlignment = Alignment.Center) {
                    Icon(Icons.Default.BrokenImage, "エラー", modifier = Modifier.size(48.dp))
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

## キャッシュ設定

```kotlin
// Application.onCreate
class MyApp : Application(), SingletonImageLoader.Factory {
    override fun newImageLoader(context: Context): ImageLoader {
        return ImageLoader.Builder(context)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .memoryCache {
                MemoryCache.Builder()
                    .maxSizePercent(context, 0.25)
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizePercent(0.02)
                    .build()
            }
            .crossfade(true)
            .build()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AsyncImage` | 非同期画像読み込み |
| `SubcomposeAsyncImage` | 読み込み状態別UI |
| `ImageRequest` | リクエスト設定 |
| `MemoryCache/DiskCache` | キャッシュ制御 |

- `AsyncImage`でURLから直接画像表示
- `crossfade(true)`でフェードインアニメーション
- `SubcomposeAsyncImage`で読み込み/エラー状態のUI分岐
- メモリ+ディスクキャッシュで効率的な画像管理

---

8種類のAndroidアプリテンプレート（画像対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camerax-analysis-2026)
- [Firebase Storage](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-storage-2026)
- [LazyGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-grid-2026)
