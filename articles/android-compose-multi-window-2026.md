---
title: "マルチウィンドウ完全ガイド — 分割画面/フリーフォーム/ドラッグ&ドロップ"
emoji: "🪟"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

**マルチウィンドウ**（分割画面、フリーフォームモード、ウィンドウ間ドラッグ&ドロップ）を解説します。

---

## マルチウィンドウ検知

```kotlin
@Composable
fun MultiWindowAwareScreen() {
    val activity = LocalContext.current as Activity
    val isInMultiWindowMode = activity.isInMultiWindowMode

    val configuration = LocalConfiguration.current
    val screenWidth = configuration.screenWidthDp.dp

    if (isInMultiWindowMode) {
        // 分割画面: コンパクトレイアウト
        CompactLayout()
    } else if (screenWidth > 600.dp) {
        // フルスクリーン + ワイド: 2ペイン
        TwoPaneLayout()
    } else {
        // 通常スマホ
        SinglePaneLayout()
    }
}

@Composable
fun CompactLayout() {
    Column(Modifier.fillMaxSize().padding(8.dp)) {
        Text("コンパクトモード", style = MaterialTheme.typography.titleSmall)
        LazyColumn {
            items(20) { index ->
                ListItem(headlineContent = { Text("アイテム $index") })
            }
        }
    }
}
```

---

## ライフサイクル対応

```kotlin
// Manifest: 分割画面対応
// <activity android:resizeableActivity="true" />

class MultiWindowActivity : ComponentActivity() {
    override fun onMultiWindowModeChanged(isInMultiWindowMode: Boolean, newConfig: Configuration) {
        super.onMultiWindowModeChanged(isInMultiWindowMode, newConfig)
        // マルチウィンドウモード変更時の処理
    }

    override fun onConfigurationChanged(newConfig: Configuration) {
        super.onConfigurationChanged(newConfig)
        // 画面サイズ変更に対応
    }
}

// Compose: ウィンドウサイズ変化の監視
@Composable
fun WindowSizeMonitor() {
    val configuration = LocalConfiguration.current

    // configurationが変わると自動で再コンポーズ
    val isCompact = configuration.screenWidthDp < 400

    if (isCompact) {
        Text("小さいウィンドウ")
    } else {
        Text("十分なウィンドウサイズ")
    }
}
```

---

## ウィンドウ間データ共有

```kotlin
@Composable
fun DragSourceContent() {
    val context = LocalContext.current

    Text(
        "ドラッグして共有",
        modifier = Modifier
            .pointerInput(Unit) {
                detectDragGesturesAfterLongPress(
                    onDragStart = {
                        val clipData = ClipData.newPlainText("text", "共有テキスト")
                        val shadow = View.DragShadowBuilder()
                        (context as Activity).startDragAndDrop(
                            clipData, shadow, null,
                            View.DRAG_FLAG_GLOBAL or View.DRAG_FLAG_GLOBAL_URI_READ
                        )
                    },
                    onDrag = { _, _ -> }
                )
            }
            .padding(16.dp)
    )
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 分割画面検知 | `isInMultiWindowMode` |
| サイズ変更 | `onConfigurationChanged` |
| リサイズ許可 | `resizeableActivity` |
| データ共有 | `startDragAndDrop` |

- `isInMultiWindowMode`でマルチウィンドウを検知
- `LocalConfiguration`でウィンドウサイズに応じたレイアウト
- `resizeableActivity="true"`で分割画面対応
- ウィンドウ間ドラッグ&ドロップでデータ共有

---

8種類のAndroidアプリテンプレート（マルチウィンドウ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [横画面/タブレット](https://zenn.dev/myougatheaxo/articles/android-compose-landscape-tablet-2026)
- [折りたたみデバイス](https://zenn.dev/myougatheaxo/articles/android-compose-foldable-device-2026)
- [Material3 Adaptive](https://zenn.dev/myougatheaxo/articles/android-compose-material3-adaptive-2026)
