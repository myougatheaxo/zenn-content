---
title: "Predictive Back完全ガイド — Android 14予測型戻るジェスチャー/Compose対応"
emoji: "👈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**Predictive Back**（予測型戻るジェスチャー、カスタムアニメーション、Navigation連携、BottomSheet対応）を解説します。

---

## Predictive Backとは

Android 14で導入された予測型戻るジェスチャー。戻る操作中にプレビューを表示し、完了前にユーザーがキャンセル可能です。

```xml
<!-- AndroidManifest.xml -->
<application
    android:enableOnBackInvokedCallback="true">
</application>
```

---

## 基本対応

```kotlin
// Compose Navigation は自動対応
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home",
        // Predictive Backアニメーションが自動適用
        enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Start) },
        exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Start) },
        popEnterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.End) },
        popExitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.End) }
    ) {
        composable("home") { HomeScreen(navController) }
        composable("detail") { DetailScreen() }
    }
}
```

---

## カスタムPredictive Back

```kotlin
@Composable
fun PredictiveBackSheet(
    onDismiss: () -> Unit,
    content: @Composable () -> Unit
) {
    var progress by remember { mutableFloatStateOf(0f) }

    PredictiveBackHandler(enabled = true) { backEvent ->
        backEvent.collect { event ->
            progress = event.progress
        }
        onDismiss()
        progress = 0f
    }

    val scale = 1f - (progress * 0.1f)
    val alpha = 1f - progress

    Box(
        Modifier
            .fillMaxSize()
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                this.alpha = alpha
            }
    ) {
        content()
    }
}
```

---

## BackHandler（従来の戻る処理）

```kotlin
@Composable
fun EditScreen(onNavigateBack: () -> Unit) {
    var hasUnsavedChanges by remember { mutableStateOf(false) }
    var showDialog by remember { mutableStateOf(false) }

    BackHandler(enabled = hasUnsavedChanges) {
        showDialog = true
    }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("変更を破棄しますか？") },
            confirmButton = {
                TextButton(onClick = { onNavigateBack() }) { Text("破棄") }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) { Text("キャンセル") }
            }
        )
    }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = "",
            onValueChange = { hasUnsavedChanges = true },
            label = { Text("入力") }
        )
    }
}
```

---

## ModalBottomSheet対応

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PredictiveBackBottomSheet() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()

    Button(onClick = { showSheet = true }) { Text("シートを開く") }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
            // Material3のModalBottomSheetはPredictive Back自動対応
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("BottomSheet Content")
                Spacer(Modifier.height(32.dp))
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 有効化 | `enableOnBackInvokedCallback=true` |
| Navigation | 自動対応（アニメーション付き） |
| カスタム | `PredictiveBackHandler` |
| 従来の戻る | `BackHandler` |
| BottomSheet | Material3自動対応 |

- `enableOnBackInvokedCallback`でPredictive Back有効化
- Compose Navigationは自動でプレビューアニメーション
- `PredictiveBackHandler`でカスタムアニメーション
- Material3コンポーネントは多くが自動対応

---

8種類のAndroidアプリテンプレート（Predictive Back対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-dialog-2026)
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-gesture-touch-2026)
