---
title: "折りたたみデバイス対応完全ガイド — WindowLayoutInfo/ヒンジ検知/Foldレイアウト"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

**折りたたみデバイス対応**（WindowLayoutInfo、ヒンジ検知、FoldingFeature、適応レイアウト）を解説します。

---

## WindowLayoutInfo取得

```kotlin
@Composable
fun FoldableAwareLayout(activity: Activity) {
    val layoutInfo by WindowInfoTracker.getOrCreate(activity)
        .windowLayoutInfo(activity)
        .collectAsStateWithLifecycle(WindowLayoutInfo(emptyList()))

    val foldingFeature = layoutInfo.displayFeatures
        .filterIsInstance<FoldingFeature>()
        .firstOrNull()

    when {
        foldingFeature?.state == FoldingFeature.State.HALF_OPENED &&
        foldingFeature.orientation == FoldingFeature.Orientation.HORIZONTAL -> {
            // テーブルトップモード（上半分表示、下半分コントロール）
            TableTopLayout()
        }
        foldingFeature?.state == FoldingFeature.State.HALF_OPENED &&
        foldingFeature.orientation == FoldingFeature.Orientation.VERTICAL -> {
            // ブックモード（左右分割）
            BookLayout()
        }
        foldingFeature?.state == FoldingFeature.State.FLAT -> {
            // 完全に開いた状態
            ExpandedLayout()
        }
        else -> {
            // 折りたたみ/通常デバイス
            CompactLayout()
        }
    }
}
```

---

## テーブルトップモード

```kotlin
@Composable
fun TableTopLayout() {
    Column(Modifier.fillMaxSize()) {
        // 上半分: コンテンツ表示
        Box(Modifier.weight(1f).fillMaxWidth()) {
            VideoPlayer(modifier = Modifier.fillMaxSize())
        }

        HorizontalDivider()

        // 下半分: コントロール
        Box(Modifier.weight(1f).fillMaxWidth().padding(16.dp)) {
            Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                Text("再生コントロール", style = MaterialTheme.typography.titleMedium)
                Row(
                    Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceEvenly
                ) {
                    IconButton(onClick = {}) { Icon(Icons.Default.SkipPrevious, "前へ") }
                    IconButton(onClick = {}) { Icon(Icons.Default.PlayArrow, "再生") }
                    IconButton(onClick = {}) { Icon(Icons.Default.SkipNext, "次へ") }
                }
                Slider(value = 0.5f, onValueChange = {})
            }
        }
    }
}
```

---

## ヒンジ回避レイアウト

```kotlin
@Composable
fun HingeAwareRow(
    foldingFeature: FoldingFeature?,
    left: @Composable () -> Unit,
    right: @Composable () -> Unit
) {
    if (foldingFeature != null &&
        foldingFeature.orientation == FoldingFeature.Orientation.VERTICAL
    ) {
        val hingeBounds = foldingFeature.bounds
        Row(Modifier.fillMaxSize()) {
            Box(Modifier.width(with(LocalDensity.current) { hingeBounds.left.toDp() })) {
                left()
            }
            Spacer(Modifier.width(with(LocalDensity.current) { hingeBounds.width().toDp() }))
            Box(Modifier.fillMaxWidth()) {
                right()
            }
        }
    } else {
        Row(Modifier.fillMaxSize()) {
            Box(Modifier.weight(1f)) { left() }
            Box(Modifier.weight(1f)) { right() }
        }
    }
}
```

---

## まとめ

| 状態 | レイアウト |
|------|-----------|
| HALF_OPENED + HORIZONTAL | テーブルトップ |
| HALF_OPENED + VERTICAL | ブック |
| FLAT | 展開（2ペイン） |
| 折りたたみ | コンパクト |

- `WindowInfoTracker`で折りたたみ状態をリアルタイム検知
- テーブルトップモードで上下分割レイアウト
- ヒンジ領域を避けたコンテンツ配置
- `FoldingFeature`で状態と向きを判定

---

8種類のAndroidアプリテンプレート（折りたたみ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [横画面/タブレット](https://zenn.dev/myougatheaxo/articles/android-compose-landscape-tablet-2026)
- [Material3 Adaptive](https://zenn.dev/myougatheaxo/articles/android-compose-material3-adaptive-2026)
- [WindowInsets](https://zenn.dev/myougatheaxo/articles/android-compose-window-insets-2026)
