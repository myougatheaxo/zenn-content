---
title: "Compose Foldable完全ガイド — 折りたたみデバイス/ヒンジ検出/Fold対応レイアウト"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "foldable"]
published: true
---

## この記事で学べること

**Compose Foldable**（折りたたみデバイス対応、ヒンジ検出、FoldingFeature、適応レイアウト）を解説します。

---

## FoldingFeature検出

```kotlin
// build.gradle
// implementation "androidx.window:window:1.2.0"

@Composable
fun FoldableAwareScreen() {
    val context = LocalContext.current
    val activity = context as Activity
    var foldingFeature by remember { mutableStateOf<FoldingFeature?>(null) }

    LaunchedEffect(Unit) {
        WindowInfoTracker.getOrCreate(activity)
            .windowLayoutInfo(activity)
            .collect { layoutInfo ->
                foldingFeature = layoutInfo.displayFeatures
                    .filterIsInstance<FoldingFeature>()
                    .firstOrNull()
            }
    }

    when {
        foldingFeature?.state == FoldingFeature.State.HALF_OPENED &&
        foldingFeature?.orientation == FoldingFeature.Orientation.HORIZONTAL -> {
            // テーブルトップモード（上半分:コンテンツ、下半分:コントロール）
            TableTopLayout()
        }
        foldingFeature?.state == FoldingFeature.State.HALF_OPENED -> {
            // ブックモード（左右分割）
            BookLayout()
        }
        else -> {
            // 通常レイアウト
            StandardLayout()
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
        // 上半分: メインコンテンツ
        Box(Modifier.weight(1f).fillMaxWidth()) {
            AsyncImage(model = "https://example.com/video-preview.jpg",
                contentDescription = null, Modifier.fillMaxSize(), contentScale = ContentScale.Crop)
        }

        // 下半分: コントロール
        Column(Modifier.weight(1f).fillMaxWidth().padding(16.dp),
            verticalArrangement = Arrangement.Center, horizontalAlignment = Alignment.CenterHorizontally) {
            Text("動画コントロール", style = MaterialTheme.typography.titleMedium)
            Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
                IconButton(onClick = {}) { Icon(Icons.Default.SkipPrevious, "前") }
                IconButton(onClick = {}) { Icon(Icons.Default.PlayArrow, "再生", Modifier.size(48.dp)) }
                IconButton(onClick = {}) { Icon(Icons.Default.SkipNext, "次") }
            }
        }
    }
}
```

---

## ブックモード

```kotlin
@Composable
fun BookLayout() {
    Row(Modifier.fillMaxSize()) {
        // 左ページ
        Box(Modifier.weight(1f).fillMaxHeight().padding(16.dp)) {
            LazyColumn { items(20) { Text("左ページ アイテム $it", Modifier.padding(8.dp)) } }
        }

        // ヒンジ部分のスペーサー
        Spacer(Modifier.width(8.dp))

        // 右ページ
        Box(Modifier.weight(1f).fillMaxHeight().padding(16.dp)) {
            Text("詳細情報", style = MaterialTheme.typography.headlineSmall)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `WindowInfoTracker` | ウィンドウ情報 |
| `FoldingFeature` | 折りたたみ状態 |
| `HALF_OPENED` | 半開き状態 |
| `Orientation` | ヒンジの向き |

- `WindowInfoTracker`で折りたたみ状態をリアルタイム監視
- テーブルトップモード: 水平ヒンジ（上:表示、下:操作）
- ブックモード: 垂直ヒンジ（左右分割）
- `WindowSizeClass`と組み合わせて完全対応

---

8種類のAndroidアプリテンプレート（折りたたみデバイス対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose WindowSizeClass](https://zenn.dev/myougatheaxo/articles/android-compose-compose-window-size-class-2026)
- [Compose TwoPane](https://zenn.dev/myougatheaxo/articles/android-compose-compose-two-pane-2026)
- [Compose MultiWindow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-multi-window-2026)
