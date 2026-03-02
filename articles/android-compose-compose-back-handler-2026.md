---
title: "BackHandler完全ガイド — BackHandler/PredictiveBack/カスタム戻るアクション"
emoji: "⬅️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**BackHandler**（BackHandler、PredictiveBackHandler、カスタム戻るアクション）を解説します。

---

## 基本BackHandler

```kotlin
@Composable
fun BackHandlerExample() {
    var showDialog by remember { mutableStateOf(false) }
    var hasUnsavedChanges by remember { mutableStateOf(false) }

    // 未保存変更がある場合にカスタム戻るアクション
    BackHandler(enabled = hasUnsavedChanges) {
        showDialog = true
    }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = "",
            onValueChange = { hasUnsavedChanges = true },
            label = { Text("テキスト入力") }
        )
    }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("変更を破棄しますか？") },
            text = { Text("保存されていない変更があります。") },
            confirmButton = {
                TextButton(onClick = { /* navigate back */ }) { Text("破棄") }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) { Text("編集を続ける") }
            }
        )
    }
}
```

---

## BottomSheet連携

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SheetBackHandler() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()
    val scope = rememberCoroutineScope()

    Button(onClick = { showSheet = true }) { Text("シートを開く") }

    if (showSheet) {
        BackHandler {
            scope.launch {
                sheetState.hide()
                showSheet = false
            }
        }

        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Text("シートコンテンツ", Modifier.padding(16.dp))
        }
    }
}
```

---

## ダブルタップで終了

```kotlin
@Composable
fun DoubleBackToExit() {
    val context = LocalContext.current
    var lastBackPress by remember { mutableLongStateOf(0L) }

    BackHandler {
        val now = System.currentTimeMillis()
        if (now - lastBackPress < 2000) {
            (context as? Activity)?.finish()
        } else {
            lastBackPress = now
            Toast.makeText(context, "もう一度押すと終了します", Toast.LENGTH_SHORT).show()
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BackHandler` | 戻るボタン処理 |
| `enabled` | 有効/無効切り替え |
| `PredictiveBackHandler` | 予測的戻るジェスチャー |
| 条件付きBack | 未保存確認 |

- `BackHandler`でシステム戻るボタンをカスタム
- `enabled`で条件付き有効化
- 未保存変更の確認ダイアログに最適
- ダブルタップ終了パターンの実装

---

8種類のAndroidアプリテンプレート（ナビゲーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-2026)
- [Navigation Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-animation-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
