---
title: "LaunchedEffect完全ガイド — 非同期処理/key制御/タイマー/初期化"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**LaunchedEffect**（非同期処理実行、key制御、タイマー、初期化パターン）を解説します。

---

## LaunchedEffect基本

```kotlin
@Composable
fun LaunchedEffectBasic(viewModel: MyViewModel = hiltViewModel()) {
    // 初回のみ実行
    LaunchedEffect(Unit) {
        viewModel.loadData()
    }

    // keyが変わるたびに再実行（前のジョブはキャンセル）
    var userId by remember { mutableStateOf(1L) }
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }
}
```

---

## タイマー/カウントダウン

```kotlin
@Composable
fun CountdownTimer(totalSeconds: Int, onFinish: () -> Unit) {
    var remaining by remember { mutableIntStateOf(totalSeconds) }

    LaunchedEffect(totalSeconds) {
        remaining = totalSeconds
        while (remaining > 0) {
            delay(1000)
            remaining--
        }
        onFinish()
    }

    Text(
        "%02d:%02d".format(remaining / 60, remaining % 60),
        style = MaterialTheme.typography.displayLarge
    )
}

// ポーリング
@Composable
fun PollingData(viewModel: DataViewModel = hiltViewModel()) {
    LaunchedEffect(Unit) {
        while (true) {
            viewModel.refresh()
            delay(30_000) // 30秒ごと
        }
    }
}
```

---

## Snackbar表示

```kotlin
@Composable
fun SnackbarOnError(errorMessage: String?, snackbarHostState: SnackbarHostState) {
    LaunchedEffect(errorMessage) {
        errorMessage?.let {
            val result = snackbarHostState.showSnackbar(
                message = it,
                actionLabel = "再試行",
                duration = SnackbarDuration.Long
            )
            if (result == SnackbarResult.ActionPerformed) {
                // 再試行
            }
        }
    }
}
```

---

## まとめ

| パターン | key | 用途 |
|---------|-----|------|
| 初回実行 | `Unit` | データロード |
| 値変更時 | 変数 | 再フェッチ |
| タイマー | `Unit` | カウントダウン |
| エラー表示 | errorMessage | Snackbar |

- `LaunchedEffect(Unit)`で初回のみ実行
- keyが変わると前のコルーチンをキャンセルして再実行
- `delay`でタイマー/ポーリング実装
- Snackbar表示のトリガーに最適

---

8種類のAndroidアプリテンプレート（非同期処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DisposableEffect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-disposable-effect-2026)
- [Side Effect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effect-2026)
- [Effect Handler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-effect-handler-2026)
