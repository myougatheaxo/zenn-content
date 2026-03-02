---
title: "Compose副作用完全ガイド — LaunchedEffect/SideEffect/DisposableEffect/derivedStateOf"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "compose"]
published: true
---

## この記事で学べること

**Compose副作用API**（LaunchedEffect、SideEffect、DisposableEffect、rememberUpdatedState、derivedStateOf）を解説します。

---

## LaunchedEffect

```kotlin
@Composable
fun TimerScreen() {
    var seconds by remember { mutableIntStateOf(0) }

    // キーが変わるとキャンセル→再起動
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            seconds++
        }
    }

    Text("経過: ${seconds}秒", style = MaterialTheme.typography.displayMedium)
}

// キー付きLaunchedEffect
@Composable
fun UserProfile(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }

    LaunchedEffect(userId) {  // userIdが変わると再実行
        user = fetchUser(userId)
    }

    user?.let { Text(it.name) }
}
```

---

## DisposableEffect

```kotlin
@Composable
fun LifecycleObserverEffect(onResume: () -> Unit, onPause: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    val currentOnResume by rememberUpdatedState(onResume)
    val currentOnPause by rememberUpdatedState(onPause)

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> currentOnResume()
                Lifecycle.Event.ON_PAUSE -> currentOnPause()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

---

## derivedStateOf / snapshotFlow

```kotlin
@Composable
fun FilteredList(items: List<Item>) {
    var query by remember { mutableStateOf("") }
    val listState = rememberLazyListState()

    // 派生状態（不要な再コンポーズを防ぐ）
    val filteredItems by remember(items) {
        derivedStateOf {
            if (query.isEmpty()) items
            else items.filter { it.name.contains(query, ignoreCase = true) }
        }
    }

    // スクロール位置の監視
    val showScrollToTop by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }

    // snapshotFlowで状態変化をFlowに変換
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                // スクロール位置変化時の処理
            }
    }

    Column {
        OutlinedTextField(value = query, onValueChange = { query = it }, label = { Text("検索") })
        LazyColumn(state = listState) {
            items(filteredItems) { item -> Text(item.name) }
        }
        if (showScrollToTop) {
            FloatingActionButton(onClick = { /* scroll to top */ }) {
                Icon(Icons.Default.ArrowUpward, null)
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LaunchedEffect` | コルーチン起動 |
| `DisposableEffect` | リソース管理 |
| `SideEffect` | 毎コンポーズ時実行 |
| `derivedStateOf` | 派生状態計算 |
| `snapshotFlow` | State→Flow変換 |

- `LaunchedEffect`でキー変化時にコルーチン再起動
- `DisposableEffect`でリスナー登録/解除を自動管理
- `rememberUpdatedState`でラムダの最新値を保持
- `derivedStateOf`で不要な再コンポーズを防止

---

8種類のAndroidアプリテンプレート（Compose最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [再コンポーズ](https://zenn.dev/myougatheaxo/articles/android-compose-compose-recomposition-2026)
- [Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-lifecycle-aware-2026)
