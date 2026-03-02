---
title: "Snackbar完全ガイド — SnackbarHost/アクション付き/カスタムSnackbar"
emoji: "🍫"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Snackbar**（SnackbarHost、アクション付きSnackbar、カスタムSnackbar、Undoパターン）を解説します。

---

## 基本Snackbar

```kotlin
@Composable
fun SnackbarExample() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        Column(Modifier.padding(padding).padding(16.dp)) {
            Button(onClick = {
                scope.launch {
                    snackbarHostState.showSnackbar("保存しました")
                }
            }) { Text("保存") }

            Spacer(Modifier.height(8.dp))

            Button(onClick = {
                scope.launch {
                    val result = snackbarHostState.showSnackbar(
                        message = "アイテムを削除しました",
                        actionLabel = "元に戻す",
                        duration = SnackbarDuration.Short
                    )
                    if (result == SnackbarResult.ActionPerformed) {
                        // Undo処理
                    }
                }
            }) { Text("削除（Undo付き）") }
        }
    }
}
```

---

## カスタムSnackbar

```kotlin
@Composable
fun CustomSnackbarHost(hostState: SnackbarHostState) {
    SnackbarHost(hostState) { data ->
        Card(
            modifier = Modifier.padding(16.dp).fillMaxWidth(),
            shape = RoundedCornerShape(12.dp),
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.inverseSurface
            )
        ) {
            Row(
                Modifier.padding(12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Icon(Icons.Default.CheckCircle, null, tint = Color.Green)
                Spacer(Modifier.width(8.dp))
                Text(
                    data.visuals.message,
                    color = MaterialTheme.colorScheme.inverseOnSurface,
                    modifier = Modifier.weight(1f)
                )
                data.visuals.actionLabel?.let { label ->
                    TextButton(onClick = { data.performAction() }) {
                        Text(label, color = MaterialTheme.colorScheme.primary)
                    }
                }
            }
        }
    }
}
```

---

## ViewModelからSnackbar表示

```kotlin
@HiltViewModel
class ItemViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    private val _snackbarEvent = Channel<String>()
    val snackbarEvent = _snackbarEvent.receiveAsFlow()

    fun deleteItem(item: Item) {
        viewModelScope.launch {
            repository.delete(item)
            _snackbarEvent.send("${item.name}を削除しました")
        }
    }
}

@Composable
fun ItemScreen(viewModel: ItemViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.snackbarEvent.collect { message ->
            snackbarHostState.showSnackbar(message)
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        // content
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SnackbarHostState` | Snackbar表示制御 |
| `showSnackbar` | メッセージ表示 |
| `actionLabel` | アクションボタン |
| `SnackbarHost` | カスタム表示 |

- `SnackbarHostState`で表示をコルーチンで制御
- `actionLabel`でUndo等のアクション付きSnackbar
- カスタム`SnackbarHost`でデザイン変更
- ViewModelから`Channel`経由でイベント送信

---

8種類のAndroidアプリテンプレート（UI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [Bottom Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
- [Side Effects](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effects-2026)
