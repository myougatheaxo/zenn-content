---
title: "Compose PredictiveBack完全ガイド — 予測型戻るジェスチャー/BackHandler/アニメーション"
emoji: "👈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**Compose PredictiveBack**（予測型戻るジェスチャー、PredictiveBackHandler、カスタムアニメーション）を解説します。

---

## 基本BackHandler

```kotlin
@Composable
fun BackHandlerDemo() {
    var showDialog by remember { mutableStateOf(false) }
    var hasUnsavedChanges by remember { mutableStateOf(false) }

    // 変更がある場合のみ戻るを横取り
    BackHandler(enabled = hasUnsavedChanges) {
        showDialog = true
    }

    Column(Modifier.padding(16.dp)) {
        TextField(
            value = "",
            onValueChange = { hasUnsavedChanges = true },
            label = { Text("入力") }
        )
    }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("確認") },
            text = { Text("変更を破棄しますか？") },
            confirmButton = { TextButton(onClick = { /* navigateBack */ }) { Text("破棄") } },
            dismissButton = { TextButton(onClick = { showDialog = false }) { Text("キャンセル") } }
        )
    }
}
```

---

## PredictiveBackHandler

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PredictiveBackDemo(onBack: () -> Unit) {
    var progress by remember { mutableFloatStateOf(0f) }

    PredictiveBackHandler(true) { backEvent ->
        backEvent.collect { event ->
            progress = event.progress
        }
        onBack()
        progress = 0f
    }

    Box(
        Modifier
            .fillMaxSize()
            .graphicsLayer {
                val scale = 1f - (progress * 0.1f)
                scaleX = scale
                scaleY = scale
                alpha = 1f - (progress * 0.3f)
            }
    ) {
        // コンテンツ
        Card(Modifier.fillMaxSize().padding(16.dp)) {
            Text("スワイプで戻る", Modifier.padding(16.dp))
        }
    }
}
```

---

## BottomSheet + PredictiveBack

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SheetWithPredictiveBack() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            BackHandler {
                showSheet = false
            }
            Column(Modifier.padding(16.dp)) {
                Text("シート内容", style = MaterialTheme.typography.titleLarge)
                Spacer(Modifier.height(100.dp))
            }
        }
    }

    Button(onClick = { showSheet = true }) { Text("シートを開く") }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BackHandler` | 戻るボタン制御 |
| `PredictiveBackHandler` | 予測型戻りジェスチャー |
| `progress` | ジェスチャー進行度 |
| `graphicsLayer` | 進行度アニメーション |

- `BackHandler`で戻るボタン/ジェスチャーを横取り
- `PredictiveBackHandler`で進行度に応じたアニメーション
- Android 14以降で予測型戻りジェスチャー対応
- `enabled`パラメータで条件付き有効化

---

8種類のAndroidアプリテンプレート（最新API対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose SharedElement](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shared-element-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
- [Navigation TypeSafe](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-type-safe-2026)
