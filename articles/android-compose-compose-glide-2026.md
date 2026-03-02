---
title: "Compose Glide完全ガイド — GlideImage/Compose統合/キャッシュ/変換"
emoji: "🏞️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "glide"]
published: true
---

## この記事で学べること

**Compose Glide**（GlideImage、Compose統合、キャッシュ設定、画像変換）を解説します。

---

## 基本GlideImage

```kotlin
// build.gradle
// implementation "com.github.bumptech.glide:compose:1.0.0-beta01"
// ksp "com.github.bumptech.glide:ksp:4.16.0"

@Composable
fun GlideDemo() {
    GlideImage(
        model = "https://example.com/image.jpg",
        contentDescription = "サンプル画像",
        modifier = Modifier.fillMaxWidth().height(200.dp),
        contentScale = ContentScale.Crop
    )
}

// プレースホルダー+エラー
@Composable
fun GlideWithFallback(url: String) {
    GlideImage(
        model = url,
        contentDescription = null,
        modifier = Modifier.size(100.dp).clip(RoundedCornerShape(8.dp)),
        contentScale = ContentScale.Crop
    ) {
        it.placeholder(R.drawable.placeholder)
            .error(R.drawable.error_image)
            .transition(DrawableTransitionOptions.withCrossFade())
    }
}
```

---

## 円形画像

```kotlin
@Composable
fun CircularAvatar(url: String, size: Dp = 64.dp) {
    GlideImage(
        model = url,
        contentDescription = "アバター",
        modifier = Modifier
            .size(size)
            .clip(CircleShape)
            .border(2.dp, MaterialTheme.colorScheme.primary, CircleShape),
        contentScale = ContentScale.Crop
    ) {
        it.circleCrop()
    }
}

@Composable
fun UserListItem(user: User) {
    ListItem(
        headlineContent = { Text(user.name) },
        supportingContent = { Text(user.email) },
        leadingContent = { CircularAvatar(user.avatarUrl, 48.dp) }
    )
}
```

---

## GlideModule設定

```kotlin
@GlideModule
class MyGlideModule : AppGlideModule() {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        builder.setMemoryCache(LruResourceCache(20 * 1024 * 1024)) // 20MB
        builder.setDiskCache(InternalCacheDiskCacheFactory(context, "glide_cache", 100 * 1024 * 1024)) // 100MB
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GlideImage` | Compose画像表示 |
| `circleCrop()` | 円形切り抜き |
| `withCrossFade()` | フェードアニメーション |
| `GlideModule` | グローバル設定 |

- `GlideImage`でCompose内でGlideを使用
- ラムダでGlideのRequestBuilder設定を適用
- `circleCrop()`/`centerCrop()`等のTransformation利用可能
- Coilと比較して大規模プロジェクトで安定

---

8種類のAndroidアプリテンプレート（画像読み込み対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Coil](https://zenn.dev/myougatheaxo/articles/android-compose-compose-coil-2026)
- [Compose ImageCropper](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-cropper-2026)
- [Compose Shimmer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shimmer-2026)
