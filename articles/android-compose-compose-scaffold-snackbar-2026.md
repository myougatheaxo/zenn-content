---
title: "Compose Scaffold+Snackbar完全ガイド — SnackbarHost/アクション付き/カスタムSnackbar"
emoji: "📢"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Scaffold+Snackbar**（SnackbarHost、SnackbarHostState、アクション付きSnackbar、カスタムSnackbar）を解説します。

---

## 基本Snackbar

```kotlin
@Composable
fun SnackbarDemo() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(Modifier.padding(padding).padding(16.dp)) {
            Button(onClick = {
                scope.launch {
                    snackbarHostState.showSnackbar("保存しました")
                }
            }) {
                Text("Snackbar表示")
            }
        }
    }
}
```

---

## アクション付きSnackbar

```kotlin
@Composable
fun ActionSnackbarDemo() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
    var items by remember { mutableStateOf(listOf("A", "B", "C")) }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(items) { item ->
                ListItem(
                    headlineContent = { Text(item) },
                    trailingContent = {
                        IconButton(onClick = {
                            val removed = item
                            items = items - removed
                            scope.launch {
                                val result = snackbarHostState.showSnackbar(
                                    message = "${removed}を削除しました",
                                    actionLabel = "元に戻す",
                                    duration = SnackbarDuration.Short
                                )
                                if (result == SnackbarResult.ActionPerformed) {
                                    items = items + removed
                                }
                            }
                        }) { Icon(Icons.Default.Delete, "削除") }
                    }
                )
            }
        }
    }
}
```

---

## カスタムSnackbar

```kotlin
@Composable
fun CustomSnackbarDemo() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = {
            SnackbarHost(snackbarHostState) { data ->
                Snackbar(
                    snackbarData = data,
                    containerColor = MaterialTheme.colorScheme.inverseSurface,
                    contentColor = MaterialTheme.colorScheme.inverseOnSurface,
                    actionColor = MaterialTheme.colorScheme.inversePrimary,
                    shape = RoundedCornerShape(12.dp)
                )
            }
        }
    ) { padding ->
        Column(Modifier.padding(padding).padding(16.dp)) {
            Button(onClick = {
                scope.launch {
                    snackbarHostState.showSnackbar(
                        message = "ネットワークエラー",
                        actionLabel = "再試行",
                        duration = SnackbarDuration.Long
                    )
                }
            }) { Text("エラーSnackbar") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SnackbarHostState` | Snackbar状態管理 |
| `SnackbarHost` | Scaffold内の配置 |
| `showSnackbar` | 表示（suspend） |
| `SnackbarResult` | アクション結果判定 |

- `SnackbarHostState.showSnackbar()`はsuspend関数（coroutine内で呼ぶ）
- `actionLabel`でUndo等のアクションボタン追加
- `SnackbarResult.ActionPerformed`でアクション実行を検出
- `SnackbarHost`のcontentパラメータでカスタムUI

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TopAppBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-top-app-bar-2026)
- [Compose BottomNavigation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-navigation-2026)
- [Compose ModalBottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-modal-bottom-sheet-2026)
