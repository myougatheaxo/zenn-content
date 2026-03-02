---
title: "Snackbar完全ガイド — ComposeでのSnackbarHost・アクション・カスタマイズ"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**Snackbar**をComposeで表示する方法を、`SnackbarHost`・アクション付き・カスタムデザインまで解説します。

---

## 基本のSnackbar

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
                Text("Snackbar表示")
            }
        }
    }
}
```

---

## アクション付きSnackbar

```kotlin
Button(onClick = {
    scope.launch {
        val result = snackbarHostState.showSnackbar(
            message = "タスクを削除しました",
            actionLabel = "元に戻す",
            duration = SnackbarDuration.Short
        )
        when (result) {
            SnackbarResult.ActionPerformed -> {
                // 元に戻す処理
                viewModel.undoDelete()
            }
            SnackbarResult.Dismissed -> {
                // 無視された
            }
        }
    }
}) {
    Text("削除")
}
```

---

## SnackbarDuration

```kotlin
// 表示時間の指定
snackbarHostState.showSnackbar(
    message = "メッセージ",
    duration = SnackbarDuration.Short      // 約4秒
    // duration = SnackbarDuration.Long    // 約10秒
    // duration = SnackbarDuration.Indefinite  // 手動で閉じるまで
)
```

---

## ViewModelからSnackbar

```kotlin
// ViewModel
class TaskViewModel : ViewModel() {
    private val _snackbarMessage = MutableSharedFlow<String>()
    val snackbarMessage: SharedFlow<String> = _snackbarMessage.asSharedFlow()

    fun deleteTask(task: Task) {
        viewModelScope.launch {
            repository.delete(task)
            _snackbarMessage.emit("「${task.title}」を削除しました")
        }
    }
}

// Composable
@Composable
fun TaskScreen(viewModel: TaskViewModel) {
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.snackbarMessage.collect { message ->
            snackbarHostState.showSnackbar(message)
        }
    }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { /* ... */ }
}
```

---

## カスタムSnackbar

```kotlin
SnackbarHost(snackbarHostState) { data ->
    Snackbar(
        snackbarData = data,
        containerColor = MaterialTheme.colorScheme.inverseSurface,
        contentColor = MaterialTheme.colorScheme.inverseOnSurface,
        actionColor = MaterialTheme.colorScheme.inversePrimary,
        shape = RoundedCornerShape(12.dp),
        modifier = Modifier.padding(16.dp)
    )
}
```

---

## エラーSnackbar

```kotlin
@Composable
fun ErrorSnackbar(snackbarHostState: SnackbarHostState) {
    SnackbarHost(snackbarHostState) { data ->
        Snackbar(
            containerColor = MaterialTheme.colorScheme.errorContainer,
            contentColor = MaterialTheme.colorScheme.onErrorContainer,
            actionColor = MaterialTheme.colorScheme.error,
            snackbarData = data
        )
    }
}
```

---

## まとめ

- `SnackbarHostState` + `Scaffold.snackbarHost` で表示
- `showSnackbar()` はsuspend関数 → `scope.launch`で呼ぶ
- `actionLabel` でアクションボタン追加
- `SnackbarResult` でアクション/Dismissedを判定
- `SharedFlow` でViewModelから発火
- `SnackbarHost`のlambdaでカスタムデザイン

---

8種類のAndroidアプリテンプレート（Snackbar通知設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ダイアログ完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-dialog-guide-2026)
- [BottomSheet完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
