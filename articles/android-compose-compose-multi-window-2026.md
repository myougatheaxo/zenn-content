---
title: "Compose MultiWindow完全ガイド — マルチウィンドウ/分割画面/フリーフォーム"
emoji: "🪟"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "multiwindow"]
published: true
---

## この記事で学べること

**Compose MultiWindow**（分割画面対応、マルチウィンドウ検出、レイアウト最適化）を解説します。

---

## マルチウィンドウ検出

```kotlin
@Composable
fun MultiWindowAware() {
    val activity = LocalContext.current as Activity
    val isInMultiWindowMode = activity.isInMultiWindowMode

    if (isInMultiWindowMode) {
        CompactLayout()
    } else {
        FullLayout()
    }
}

@Composable
fun CompactLayout() {
    Column(Modifier.fillMaxSize().padding(8.dp)) {
        Text("コンパクトモード", style = MaterialTheme.typography.titleMedium)
        LazyColumn { items(10) { Text("Item $it", Modifier.padding(4.dp)) } }
    }
}

@Composable
fun FullLayout() {
    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("フルモード", style = MaterialTheme.typography.headlineMedium)
        LazyColumn { items(10) { Card(Modifier.fillMaxWidth().padding(8.dp)) { Text("Item $it", Modifier.padding(16.dp)) } } }
    }
}
```

---

## WindowSizeClassと組み合わせ

```kotlin
@Composable
fun AdaptiveContent(windowSizeClass: WindowSizeClass) {
    val activity = LocalContext.current as Activity

    when {
        activity.isInMultiWindowMode -> {
            // マルチウィンドウ時は常にコンパクト
            SingleColumnLayout()
        }
        windowSizeClass.widthSizeClass == WindowWidthSizeClass.Expanded -> {
            TwoColumnLayout()
        }
        else -> {
            SingleColumnLayout()
        }
    }
}
```

---

## Drag & Drop（ウィンドウ間）

```kotlin
@Composable
fun DragDropTarget() {
    var droppedText by remember { mutableStateOf<String?>(null) }

    Box(
        Modifier
            .fillMaxSize()
            .dragAndDropTarget(
                shouldStartDragAndDrop = { event ->
                    event.clipDescription.hasMimeType(ClipDescription.MIMETYPE_TEXT_PLAIN)
                },
                target = remember {
                    object : DragAndDropTarget {
                        override fun onDrop(event: DragAndDropEvent): Boolean {
                            val item = event.toAndroidDragEvent().clipData.getItemAt(0)
                            droppedText = item.text.toString()
                            return true
                        }
                    }
                }
            ),
        contentAlignment = Alignment.Center
    ) {
        Text(droppedText ?: "ここにドロップ", style = MaterialTheme.typography.titleMedium)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `isInMultiWindowMode` | マルチウィンドウ検出 |
| `WindowSizeClass` | サイズ分類 |
| `dragAndDropTarget` | ウィンドウ間D&D |
| `configChanges` | 設定変更対応 |

- `isInMultiWindowMode`でレイアウト切替
- マルチウィンドウ時はコンパクトレイアウト推奨
- `WindowSizeClass`との組み合わせで最適化
- ウィンドウ間Drag & Dropでデータ共有

---

8種類のAndroidアプリテンプレート（マルチウィンドウ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose WindowSizeClass](https://zenn.dev/myougatheaxo/articles/android-compose-compose-window-size-class-2026)
- [Compose PiP](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pip-2026)
- [Compose TwoPane](https://zenn.dev/myougatheaxo/articles/android-compose-compose-two-pane-2026)
