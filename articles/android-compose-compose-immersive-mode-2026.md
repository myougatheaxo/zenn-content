---
title: "Compose ImmersiveMode完全ガイド — フルスクリーン/没入モード/SystemUI非表示/ジェスチャーナビ"
emoji: "🖥️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "systemui"]
published: true
---

## この記事で学べること

**Compose ImmersiveMode**（フルスクリーン、没入モード、SystemUI非表示、ジェスチャーナビゲーション）を解説します。

---

## 没入モード

```kotlin
@Composable
fun ImmersiveScreen() {
    val view = LocalView.current

    DisposableEffect(Unit) {
        val window = (view.context as Activity).window
        val controller = WindowCompat.getInsetsController(window, view)

        // フルスクリーン
        controller.hide(WindowInsetsCompat.Type.systemBars())
        controller.systemBarsBehavior =
            WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE

        onDispose {
            controller.show(WindowInsetsCompat.Type.systemBars())
        }
    }

    Box(
        Modifier.fillMaxSize().background(Color.Black),
        contentAlignment = Alignment.Center
    ) {
        Text("没入モード", color = Color.White, style = MaterialTheme.typography.displayMedium)
    }
}
```

---

## 動画プレーヤー

```kotlin
@Composable
fun VideoPlayerScreen() {
    var isFullscreen by remember { mutableStateOf(false) }
    val view = LocalView.current

    LaunchedEffect(isFullscreen) {
        val window = (view.context as Activity).window
        val controller = WindowCompat.getInsetsController(window, view)

        if (isFullscreen) {
            controller.hide(WindowInsetsCompat.Type.systemBars())
            controller.systemBarsBehavior =
                WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
            // 横向き
            (view.context as Activity).requestedOrientation =
                ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE
        } else {
            controller.show(WindowInsetsCompat.Type.systemBars())
            (view.context as Activity).requestedOrientation =
                ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED
        }
    }

    Box(Modifier.fillMaxSize().background(Color.Black)) {
        // 動画コンテンツ
        Text("動画プレーヤー", color = Color.White, modifier = Modifier.align(Alignment.Center))

        // フルスクリーン切替ボタン
        IconButton(
            onClick = { isFullscreen = !isFullscreen },
            modifier = Modifier.align(Alignment.BottomEnd).padding(16.dp)
        ) {
            Icon(
                if (isFullscreen) Icons.Default.FullscreenExit else Icons.Default.Fullscreen,
                contentDescription = "フルスクリーン切替",
                tint = Color.White
            )
        }
    }
}
```

---

## タップで表示/非表示

```kotlin
@Composable
fun TapToToggleUI() {
    var showUI by remember { mutableStateOf(true) }
    val view = LocalView.current

    LaunchedEffect(showUI) {
        val controller = WindowCompat.getInsetsController(
            (view.context as Activity).window, view
        )
        if (showUI) controller.show(WindowInsetsCompat.Type.systemBars())
        else controller.hide(WindowInsetsCompat.Type.systemBars())
    }

    Box(
        Modifier.fillMaxSize()
            .background(Color.Black)
            .pointerInput(Unit) { detectTapGestures { showUI = !showUI } }
    ) {
        AsyncImage(model = "photo.jpg", contentDescription = null, modifier = Modifier.fillMaxSize())

        AnimatedVisibility(visible = showUI, modifier = Modifier.align(Alignment.TopCenter)) {
            TopAppBar(
                title = { Text("写真ビューアー") },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = Color.Black.copy(alpha = 0.5f),
                    titleContentColor = Color.White
                )
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `WindowInsetsController` | システムバー制御 |
| `hide/show` | 表示切替 |
| `BEHAVIOR_SHOW_TRANSIENT` | スワイプで一時表示 |
| `systemBars()` | ステータス+ナビバー |

- `WindowInsetsController.hide()`でシステムバーを非表示
- `BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE`でスワイプ一時表示
- `DisposableEffect`で画面離脱時にシステムバー復元
- 動画/写真ビューアーで没入体験を実現

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose StatusBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-status-bar-2026)
- [Compose GestureDetector](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-detector-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
