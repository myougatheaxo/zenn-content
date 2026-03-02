---
title: "WindowSize計算完全ガイド — calculateWindowSizeClass/BoxWithConstraints/動的UI"
emoji: "📏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

**WindowSize計算**（calculateWindowSizeClass、BoxWithConstraints、動的カラム数、レスポンシブUI）を解説します。

---

## calculateWindowSizeClass

```kotlin
@Composable
fun ResponsiveApp() {
    val windowSizeClass = calculateWindowSizeClass(LocalContext.current as Activity)

    val columns = when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> 1
        WindowWidthSizeClass.Medium -> 2
        WindowWidthSizeClass.Expanded -> 3
        else -> 1
    }

    val padding = when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> 16.dp
        WindowWidthSizeClass.Medium -> 24.dp
        WindowWidthSizeClass.Expanded -> 32.dp
        else -> 16.dp
    }

    LazyVerticalGrid(
        columns = GridCells.Fixed(columns),
        contentPadding = PaddingValues(padding)
    ) {
        items(20) { index ->
            Card(Modifier.padding(4.dp)) {
                Text("Item $index", Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## BoxWithConstraints

```kotlin
@Composable
fun AdaptiveContent() {
    BoxWithConstraints(Modifier.fillMaxSize()) {
        val isWide = maxWidth > 600.dp
        val isTall = maxHeight > 800.dp

        if (isWide) {
            Row(Modifier.fillMaxSize()) {
                SidePanel(Modifier.width(300.dp))
                VerticalDivider()
                MainContent(Modifier.weight(1f))
            }
        } else {
            Column(Modifier.fillMaxSize()) {
                MainContent(Modifier.weight(1f))
                if (!isTall) {
                    CompactBottomBar()
                }
            }
        }
    }
}

@Composable
fun ResponsiveImage() {
    BoxWithConstraints {
        val imageHeight = when {
            maxWidth > 600.dp -> 400.dp
            maxWidth > 400.dp -> 250.dp
            else -> 180.dp
        }
        AsyncImage(
            model = "https://example.com/image.jpg",
            contentDescription = null,
            modifier = Modifier.fillMaxWidth().height(imageHeight),
            contentScale = ContentScale.Crop
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `calculateWindowSizeClass` | デバイスサイズ判定 |
| `BoxWithConstraints` | 親サイズに応じた配置 |
| `GridCells.Adaptive` | 自動カラム数 |
| `LocalConfiguration` | 画面向き判定 |

- `WindowSizeClass`でCompact/Medium/Expandedを判定
- `BoxWithConstraints`で親コンテナサイズに応じた動的UI
- カラム数/パディング/レイアウト方向を動的変更
- タブレット/スマホ/折りたたみ全対応

---

8種類のAndroidアプリテンプレート（レスポンシブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [横画面/タブレット](https://zenn.dev/myougatheaxo/articles/android-compose-landscape-tablet-2026)
- [折りたたみデバイス](https://zenn.dev/myougatheaxo/articles/android-compose-foldable-device-2026)
- [Material3 Adaptive](https://zenn.dev/myougatheaxo/articles/android-compose-material3-adaptive-2026)
