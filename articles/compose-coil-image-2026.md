---
title: "Coilで画像を表示する — Composeの画像読み込みライブラリ完全ガイド"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coil"]
published: true
---

## この記事で学べること

ネットワーク画像やローカル画像をComposeで表示するなら**Coil**一択。Kotlin製で、Composeとの親和性が最高です。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.coil-kt:coil-compose:2.6.0")
}
```

---

## 基本：AsyncImage

```kotlin
AsyncImage(
    model = "https://example.com/photo.jpg",
    contentDescription = "プロフィール画像",
    modifier = Modifier.size(120.dp)
)
```

URLを渡すだけで画像が表示されます。キャッシュも自動。

---

## placeholder / error / fallback

```kotlin
AsyncImage(
    model = "https://example.com/photo.jpg",
    contentDescription = "ユーザー画像",
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error_image),
    fallback = painterResource(R.drawable.default_avatar),
    modifier = Modifier.size(80.dp)
)
```

| パラメータ | 表示タイミング |
|-----------|--------------|
| `placeholder` | 読み込み中 |
| `error` | 読み込み失敗時 |
| `fallback` | URLがnullの場合 |

---

## 丸型クリッピング

```kotlin
AsyncImage(
    model = imageUrl,
    contentDescription = "アバター",
    contentScale = ContentScale.Crop,
    modifier = Modifier
        .size(64.dp)
        .clip(CircleShape)
        .border(2.dp, MaterialTheme.colorScheme.primary, CircleShape)
)
```

`CircleShape`で丸型に切り抜き。プロフィール画像の定番パターン。

---

## ContentScale

| ContentScale | 効果 |
|-------------|------|
| `Crop` | 画像を切り抜いて領域を埋める |
| `Fit` | 画像全体を領域内に収める |
| `FillBounds` | 領域にぴったり合わせる（比率無視） |
| `Inside` | 画像が大きい場合のみ縮小 |
| `None` | 元サイズのまま |

ほとんどの場合は`Crop`か`Fit`で十分。

---

## SubcomposeAsyncImage（ローディング状態）

```kotlin
SubcomposeAsyncImage(
    model = imageUrl,
    contentDescription = "商品画像",
    modifier = Modifier
        .fillMaxWidth()
        .height(200.dp)
) {
    when (painter.state) {
        is AsyncImagePainter.State.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is AsyncImagePainter.State.Error -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                Icon(Icons.Default.BrokenImage, "エラー")
            }
        }
        else -> {
            SubcomposeAsyncImageContent()
        }
    }
}
```

ローディング中にスピナーを表示するパターン。

---

## キャッシュ制御

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .crossfade(true)           // フェードインアニメーション
        .memoryCachePolicy(CachePolicy.ENABLED)
        .diskCachePolicy(CachePolicy.ENABLED)
        .build(),
    contentDescription = null,
    modifier = Modifier.fillMaxWidth()
)
```

デフォルトでメモリキャッシュ + ディスクキャッシュが有効。明示的に制御する場合のみ設定。

---

## LazyColumnでの画像リスト

```kotlin
LazyColumn {
    items(products, key = { it.id }) { product ->
        Row(Modifier.padding(8.dp)) {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = product.name,
                contentScale = ContentScale.Crop,
                modifier = Modifier
                    .size(80.dp)
                    .clip(RoundedCornerShape(8.dp))
            )
            Spacer(Modifier.width(12.dp))
            Column {
                Text(product.name, style = MaterialTheme.typography.titleMedium)
                Text("¥${product.price}", style = MaterialTheme.typography.bodyMedium)
            }
        }
    }
}
```

`key`パラメータを指定すれば、スクロール時にキャッシュから画像が再利用されます。

---

## まとめ

- `AsyncImage`が基本（URL渡すだけ）
- `placeholder`/`error`で読み込み状態を表示
- `CircleShape`で丸型アバター
- `SubcomposeAsyncImage`でカスタムローディング
- キャッシュはデフォルトで有効

---

8種類のAndroidアプリテンプレート（画像機能の追加も容易なMVVM設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
