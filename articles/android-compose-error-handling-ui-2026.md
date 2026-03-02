---
title: "エラーハンドリングUI実装 — エラー画面/リトライ/空状態をComposeで作る"
emoji: "⚠️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "errorhandling"]
published: true
---

## この記事で学べること

Composeでの**エラー画面、リトライUI、空状態**の実装パターンを解説します。

---

## UiState パターン

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String, val retry: (() -> Unit)? = null) : UiState<Nothing>
    data object Empty : UiState<Nothing>
}

@Composable
fun <T> StatefulContent(
    state: UiState<T>,
    onRetry: () -> Unit,
    content: @Composable (T) -> Unit
) {
    when (state) {
        is UiState.Loading -> LoadingView()
        is UiState.Success -> content(state.data)
        is UiState.Error -> ErrorView(state.message, onRetry)
        is UiState.Empty -> EmptyView()
    }
}
```

---

## ローディング画面

```kotlin
@Composable
fun LoadingView(message: String = "読み込み中...") {
    Box(
        Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            CircularProgressIndicator()
            Spacer(Modifier.height(16.dp))
            Text(message, style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

---

## エラー画面

```kotlin
@Composable
fun ErrorView(
    message: String,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            Icons.Default.ErrorOutline,
            contentDescription = null,
            modifier = Modifier.size(72.dp),
            tint = MaterialTheme.colorScheme.error
        )
        Spacer(Modifier.height(16.dp))
        Text(
            "エラーが発生しました",
            style = MaterialTheme.typography.headlineSmall
        )
        Spacer(Modifier.height(8.dp))
        Text(
            message,
            style = MaterialTheme.typography.bodyMedium,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(Modifier.height(24.dp))
        Button(onClick = onRetry) {
            Icon(Icons.Default.Refresh, null)
            Spacer(Modifier.width(8.dp))
            Text("再試行")
        }
    }
}
```

---

## 空状態

```kotlin
@Composable
fun EmptyView(
    title: String = "データがありません",
    description: String = "新しい項目を追加してください",
    actionLabel: String? = null,
    onAction: (() -> Unit)? = null
) {
    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            Icons.Default.Inbox,
            contentDescription = null,
            modifier = Modifier.size(80.dp),
            tint = MaterialTheme.colorScheme.outline
        )
        Spacer(Modifier.height(16.dp))
        Text(title, style = MaterialTheme.typography.titleLarge)
        Spacer(Modifier.height(8.dp))
        Text(
            description,
            style = MaterialTheme.typography.bodyMedium,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        if (actionLabel != null && onAction != null) {
            Spacer(Modifier.height(24.dp))
            FilledTonalButton(onClick = onAction) {
                Text(actionLabel)
            }
        }
    }
}
```

---

## Snackbarエラー通知

```kotlin
@Composable
fun SnackbarErrorExample(viewModel: ItemViewModel) {
    val snackbarHostState = remember { SnackbarHostState() }
    val errorMessage by viewModel.errorMessage.collectAsStateWithLifecycle()

    LaunchedEffect(errorMessage) {
        errorMessage?.let {
            val result = snackbarHostState.showSnackbar(
                message = it,
                actionLabel = "再試行",
                duration = SnackbarDuration.Long
            )
            if (result == SnackbarResult.ActionPerformed) {
                viewModel.retry()
            }
            viewModel.clearError()
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        ItemList(Modifier.padding(padding))
    }
}
```

---

## 実践: 完全なスクリーン

```kotlin
@Composable
fun ItemListScreen(viewModel: ItemViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    StatefulContent(
        state = uiState,
        onRetry = { viewModel.loadItems() }
    ) { items ->
        if (items.isEmpty()) {
            EmptyView(
                title = "アイテムがありません",
                actionLabel = "追加する",
                onAction = { viewModel.showAddDialog() }
            )
        } else {
            LazyColumn {
                items(items, key = { it.id }) { item ->
                    ListItem(
                        headlineContent = { Text(item.title) },
                        supportingContent = { Text(item.description) }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

- `sealed interface UiState`: Loading/Success/Error/Empty
- `StatefulContent`で状態に応じたUI切り替え
- エラー画面: アイコン+メッセージ+リトライボタン
- 空状態: アイコン+説明+アクションボタン
- `Snackbar`で一時的なエラー通知+再試行
- ViewModel + StateFlowで状態管理

---

8種類のAndroidアプリテンプレート（エラーハンドリング設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [Snackbar実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-snackbar-2026)
- [Retrofit完全ガイド](https://zenn.dev/myougatheaxo/articles/android-retrofit-network-2026)
