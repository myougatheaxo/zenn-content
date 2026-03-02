---
title: "Snackbar/Toast完全ガイド — ComposeでのフィードバックUI"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Composeでの**SnackbarとToast**（表示/アクション/カスタマイズ）を解説します。

---

## Snackbar基本

```kotlin
@Composable
fun SnackbarExample() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        Column(Modifier.padding(padding)) {
            Button(onClick = {
                scope.launch {
                    snackbarHostState.showSnackbar("保存しました")
                }
            }) {
                Text("保存")
            }
        }
    }
}
```

---

## アクション付きSnackbar

```kotlin
@Composable
fun SnackbarWithAction() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        Column(Modifier.padding(padding)) {
            Button(onClick = {
                scope.launch {
                    val result = snackbarHostState.showSnackbar(
                        message = "アイテムを削除しました",
                        actionLabel = "元に戻す",
                        duration = SnackbarDuration.Short
                    )
                    when (result) {
                        SnackbarResult.ActionPerformed -> {
                            // Undo処理
                        }
                        SnackbarResult.Dismissed -> {
                            // 完全に削除
                        }
                    }
                }
            }) {
                Text("削除")
            }
        }
    }
}
```

---

## カスタムSnackbar

```kotlin
@Composable
fun CustomSnackbarHost(snackbarHostState: SnackbarHostState) {
    SnackbarHost(snackbarHostState) { data ->
        Snackbar(
            modifier = Modifier.padding(16.dp),
            containerColor = MaterialTheme.colorScheme.inverseSurface,
            contentColor = MaterialTheme.colorScheme.inverseOnSurface,
            actionColor = MaterialTheme.colorScheme.inversePrimary,
            shape = RoundedCornerShape(12.dp),
            action = {
                data.visuals.actionLabel?.let { label ->
                    TextButton(onClick = { data.performAction() }) {
                        Text(label)
                    }
                }
            }
        ) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Icon(Icons.Default.Check, null, Modifier.size(20.dp))
                Spacer(Modifier.width(8.dp))
                Text(data.visuals.message)
            }
        }
    }
}
```

---

## Toast

```kotlin
@Composable
fun ToastExample() {
    val context = LocalContext.current

    Button(onClick = {
        Toast.makeText(context, "コピーしました", Toast.LENGTH_SHORT).show()
    }) {
        Text("コピー")
    }
}
```

---

## まとめ

- `SnackbarHostState`+`Scaffold`でSnackbar表示
- `showSnackbar`のresultでアクション/ディスミスを判定
- `SnackbarDuration`で表示時間制御
- カスタムSnackbarで独自デザイン
- 簡易通知は`Toast`、操作付きは`Snackbar`
- `coroutineScope.launch`内で呼び出し

---

8種類のAndroidアプリテンプレート（フィードバックUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
- [ダイアログ/BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-modal-2026)
- [SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-swipe-dismiss-2026)
