---
title: "Android TV Compose完全ガイド — TvLazyRow/フォーカス管理/D-pad対応"
emoji: "📺"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "androidtv"]
published: true
---

## この記事で学べること

**Android TV Compose**（TvLazyRow、フォーカス管理、D-pad操作、カード表示）を解説します。

---

## TvLazyRow/Column

```kotlin
@Composable
fun TvHomeScreen(categories: List<Category>) {
    TvLazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(vertical = 24.dp)
    ) {
        items(categories) { category ->
            Text(
                category.name,
                style = MaterialTheme.typography.headlineSmall,
                modifier = Modifier.padding(start = 48.dp, bottom = 8.dp)
            )

            TvLazyRow(
                contentPadding = PaddingValues(horizontal = 48.dp),
                horizontalArrangement = Arrangement.spacedBy(16.dp)
            ) {
                items(category.items) { item ->
                    TvCard(item = item, onClick = { /* navigate */ })
                }
            }

            Spacer(Modifier.height(24.dp))
        }
    }
}
```

---

## TvCard

```kotlin
@Composable
fun TvCard(item: MediaItem, onClick: () -> Unit) {
    var isFocused by remember { mutableStateOf(false) }

    Card(
        onClick = onClick,
        modifier = Modifier
            .size(width = 200.dp, height = 120.dp)
            .onFocusChanged { isFocused = it.isFocused }
            .scale(if (isFocused) 1.1f else 1f),
        shape = RoundedCornerShape(8.dp),
        border = CardDefaults.border(
            focusedBorder = Border(
                border = BorderStroke(3.dp, MaterialTheme.colorScheme.primary),
                shape = RoundedCornerShape(8.dp)
            )
        )
    ) {
        Box(Modifier.fillMaxSize()) {
            AsyncImage(
                model = item.thumbnailUrl,
                contentDescription = item.title,
                contentScale = ContentScale.Crop,
                modifier = Modifier.fillMaxSize()
            )
            Text(
                item.title,
                modifier = Modifier
                    .align(Alignment.BottomStart)
                    .background(Color.Black.copy(alpha = 0.6f))
                    .padding(8.dp),
                color = Color.White
            )
        }
    }
}
```

---

## D-padフォーカス管理

```kotlin
@Composable
fun DpadNavigationExample() {
    val focusRequester = remember { FocusRequester() }

    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }

    Row(
        Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Button(
            onClick = {},
            modifier = Modifier
                .focusRequester(focusRequester)
                .focusable()
        ) { Text("ボタン1") }

        Button(onClick = {}, modifier = Modifier.focusable()) { Text("ボタン2") }
        Button(onClick = {}, modifier = Modifier.focusable()) { Text("ボタン3") }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|---------------|------|
| TvLazyRow | 横スクロールリスト |
| TvLazyColumn | 縦スクロールリスト |
| Card + border | フォーカス可視化 |
| FocusRequester | 初期フォーカス |

- `TvLazyRow`/`TvLazyColumn`でTV向けリスト
- フォーカス時のスケール/ボーダーでUI強調
- D-pad操作でシームレスなナビゲーション
- `FocusRequester`で初期フォーカス位置を制御

---

8種類のAndroidアプリテンプレート（TV対応可）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [横画面/タブレット](https://zenn.dev/myougatheaxo/articles/android-compose-landscape-tablet-2026)
- [Material3 Adaptive](https://zenn.dev/myougatheaxo/articles/android-compose-material3-adaptive-2026)
- [Coil画像](https://zenn.dev/myougatheaxo/articles/android-compose-coil-image-2026)
