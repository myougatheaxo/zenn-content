---
title: "マルチスクリーンサイズ対応ガイド — Compose版"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

Composeでの**マルチスクリーンサイズ対応**（dimens管理、WindowSizeClass、プレビュー確認）を解説します。

---

## WindowSizeClass判定

```kotlin
@Composable
fun rememberWindowSize(): WindowType {
    val config = LocalConfiguration.current
    return when {
        config.screenWidthDp < 600 -> WindowType.Compact
        config.screenWidthDp < 840 -> WindowType.Medium
        else -> WindowType.Expanded
    }
}

enum class WindowType { Compact, Medium, Expanded }
```

---

## レスポンシブレイアウト

```kotlin
@Composable
fun ResponsiveScreen() {
    val windowType = rememberWindowSize()

    when (windowType) {
        WindowType.Compact -> {
            // スマホ: 縦並び
            Column(Modifier.fillMaxSize()) {
                HeaderSection(Modifier.fillMaxWidth())
                ContentSection(Modifier.fillMaxWidth().weight(1f))
            }
        }
        WindowType.Medium -> {
            // タブレット小: 2カラム均等
            Row(Modifier.fillMaxSize()) {
                HeaderSection(Modifier.weight(1f))
                ContentSection(Modifier.weight(1f))
            }
        }
        WindowType.Expanded -> {
            // タブレット大: サイドバー + メイン
            Row(Modifier.fillMaxSize()) {
                HeaderSection(Modifier.width(300.dp))
                ContentSection(Modifier.weight(1f))
            }
        }
    }
}
```

---

## 動的フォントサイズ

```kotlin
@Composable
fun responsiveTextSize(): TextUnit {
    val config = LocalConfiguration.current
    return when {
        config.screenWidthDp < 360 -> 14.sp
        config.screenWidthDp < 600 -> 16.sp
        else -> 18.sp
    }
}

@Composable
fun responsivePadding(): Dp {
    val config = LocalConfiguration.current
    return when {
        config.screenWidthDp < 360 -> 8.dp
        config.screenWidthDp < 600 -> 16.dp
        else -> 24.dp
    }
}
```

---

## プレビューで確認

```kotlin
@Preview(name = "Phone", widthDp = 360, heightDp = 800)
@Preview(name = "Tablet", widthDp = 800, heightDp = 1280)
@Preview(name = "Foldable", widthDp = 600, heightDp = 900)
@Composable
fun PreviewResponsiveScreen() {
    MaterialTheme {
        ResponsiveScreen()
    }
}

// ダークモード
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Preview(name = "Light", uiMode = Configuration.UI_MODE_NIGHT_NO)
@Composable
fun PreviewTheme() {
    MyAppTheme {
        MainScreen()
    }
}
```

---

## 画面方向対応

```kotlin
@Composable
fun OrientationAwareLayout() {
    val config = LocalConfiguration.current
    val isLandscape = config.orientation == Configuration.ORIENTATION_LANDSCAPE

    if (isLandscape) {
        Row(Modifier.fillMaxSize()) {
            ImageSection(Modifier.weight(1f))
            DetailSection(Modifier.weight(1f))
        }
    } else {
        Column(Modifier.fillMaxSize()) {
            ImageSection(Modifier.fillMaxWidth().height(250.dp))
            DetailSection(Modifier.fillMaxWidth().weight(1f))
        }
    }
}
```

---

## まとめ

- `LocalConfiguration.current`で画面サイズ取得
- WindowTypeで3段階（Compact/Medium/Expanded）分岐
- 動的なfontSize/paddingでスケーリング
- `@Preview(widthDp)`で各画面サイズを確認
- 横画面は`orientation`で検出
- `BoxWithConstraints`で親コンテナサイズに応じた分岐

---

8種類のAndroidアプリテンプレート（レスポンシブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [GridLayoutガイド](https://zenn.dev/myougatheaxo/articles/android-compose-grid-layout-2026)
- [BottomNavigationガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-scaffold-2026)
- [Typography/フォント](https://zenn.dev/myougatheaxo/articles/android-compose-typography-font-2026)
