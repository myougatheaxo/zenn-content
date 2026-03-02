---
title: "Wear OS Compose入門 — ウォッチアプリ開発ガイド"
emoji: "⌚"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "wearos"]
published: true
---

## この記事で学べること

**Wear OS Compose**（ウォッチUI、ScalingLazyColumn、SwipeDismissableNavHost、ヘルスデータ）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}

android {
    compileSdk = 35
    defaultConfig {
        minSdk = 30 // Wear OS 3.0+
    }
}

dependencies {
    implementation("androidx.wear.compose:compose-material3:1.0.0-alpha34")
    implementation("androidx.wear.compose:compose-foundation:1.4.1")
    implementation("androidx.wear.compose:compose-navigation:1.4.1")
    implementation("androidx.compose.ui:ui-tooling-preview")
}
```

---

## 基本レイアウト

```kotlin
@Composable
fun WearApp() {
    MaterialTheme {
        val listState = rememberScalingLazyListState()

        Scaffold(
            timeText = { TimeText() },
            vignette = { Vignette(vignettePosition = VignettePosition.TopAndBottom) },
            positionIndicator = { PositionIndicator(scalingLazyListState = listState) }
        ) {
            ScalingLazyColumn(
                state = listState,
                modifier = Modifier.fillMaxSize(),
                anchorType = ScalingLazyListAnchorType.ItemCenter
            ) {
                item { ListHeader { Text("メニュー") } }

                item {
                    Chip(
                        onClick = { /* navigate */ },
                        label = { Text("タイマー") },
                        icon = {
                            Icon(Icons.Default.Timer, contentDescription = null)
                        },
                        colors = ChipDefaults.primaryChipColors()
                    )
                }

                item {
                    Chip(
                        onClick = { /* navigate */ },
                        label = { Text("設定") },
                        icon = {
                            Icon(Icons.Default.Settings, contentDescription = null)
                        }
                    )
                }
            }
        }
    }
}
```

---

## SwipeDismissableNavHost

```kotlin
@Composable
fun WearNavigation() {
    val navController = rememberSwipeDismissableNavController()

    SwipeDismissableNavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onTimerClick = { navController.navigate("timer") },
                onSettingsClick = { navController.navigate("settings") }
            )
        }

        composable("timer") {
            TimerScreen()
        }

        composable("settings") {
            SettingsScreen()
        }
    }
}

// スワイプで前画面に戻る（自動対応）
```

---

## Wear用ボタン・入力

```kotlin
@Composable
fun TimerScreen() {
    var seconds by remember { mutableIntStateOf(60) }
    var isRunning by remember { mutableStateOf(false) }

    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(
                text = formatTime(seconds),
                style = MaterialTheme.typography.displayLarge
            )

            Spacer(Modifier.height(8.dp))

            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                // コンパクトボタン
                CompactChip(
                    onClick = { isRunning = !isRunning },
                    label = { Text(if (isRunning) "停止" else "開始") }
                )
                CompactChip(
                    onClick = { seconds = 60; isRunning = false },
                    label = { Text("リセット") }
                )
            }
        }
    }

    // タイマーロジック
    LaunchedEffect(isRunning) {
        while (isRunning && seconds > 0) {
            delay(1000)
            seconds--
        }
        if (seconds == 0) isRunning = false
    }
}

private fun formatTime(seconds: Int): String {
    val min = seconds / 60
    val sec = seconds % 60
    return "%02d:%02d".format(min, sec)
}
```

---

## カーブテキスト・CircularProgressIndicator

```kotlin
@Composable
fun ProgressScreen(progress: Float) {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        CircularProgressIndicator(
            progress = { progress },
            modifier = Modifier.fillMaxSize().padding(4.dp),
            strokeWidth = 8.dp
        )

        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(
                text = "${(progress * 100).toInt()}%",
                style = MaterialTheme.typography.displayMedium
            )
            Text("完了", style = MaterialTheme.typography.bodySmall)
        }
    }
}

// CurvedLayout（円弧テキスト）
@Composable
fun CurvedTextExample() {
    CurvedLayout {
        curvedText(
            text = "Wear OS Compose",
            style = CurvedTextStyle(fontSize = 14.sp)
        )
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `ScalingLazyColumn` | スクロールリスト（端縮小） |
| `Scaffold` | TimeText/Vignette/PositionIndicator |
| `SwipeDismissableNavHost` | スワイプ戻りNavigation |
| `Chip`/`CompactChip` | タップ可能なアクション |
| `CircularProgressIndicator` | 円形プログレス |
| `CurvedLayout` | 円弧テキスト |

- Wear OS Composeは通常Composeと似たAPI
- `ScalingLazyColumn`で丸型画面に最適化
- スワイプナビゲーションが標準
- 画面サイズを考慮したコンパクトUI設計

---

8種類のAndroidアプリテンプレート（Wear OS対応含む）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
