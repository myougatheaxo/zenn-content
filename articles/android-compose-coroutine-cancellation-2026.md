---
title: "Coroutine Cancellation完全ガイド — キャンセル/isActive/NonCancellable"
emoji: "🚫"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Coroutine Cancellation**（キャンセル処理、isActive、CancellationException、NonCancellable）を解説します。

---

## キャンセル基本

```kotlin
@HiltViewModel
class DownloadViewModel @Inject constructor() : ViewModel() {
    private var downloadJob: Job? = null

    fun startDownload(url: String) {
        downloadJob?.cancel() // 前のダウンロードをキャンセル
        downloadJob = viewModelScope.launch {
            try {
                for (progress in 0..100 step 10) {
                    ensureActive() // キャンセルチェック
                    _progress.value = progress
                    delay(500) // delay自体もキャンセルに応答する
                }
                _status.value = "完了"
            } catch (e: CancellationException) {
                _status.value = "キャンセル済み"
                throw e // CancellationExceptionは再スロー必須
            }
        }
    }

    fun cancelDownload() {
        downloadJob?.cancel()
    }
}
```

---

## isActiveとensureActive

```kotlin
// CPU集約処理のキャンセル対応
suspend fun processLargeList(items: List<Item>): List<Result> = withContext(Dispatchers.Default) {
    items.mapIndexed { index, item ->
        // 定期的にキャンセルチェック
        if (index % 100 == 0) ensureActive()
        item.process()
    }
}

// isActiveでループ制御
suspend fun pollData() = withContext(Dispatchers.IO) {
    while (isActive) { // キャンセル時にfalseになる
        val data = fetchLatestData()
        // ...
        delay(5000)
    }
}
```

---

## NonCancellable（クリーンアップ）

```kotlin
suspend fun saveAndCleanup(data: Data) {
    try {
        processData(data)
    } finally {
        // キャンセル後でもクリーンアップは完了させる
        withContext(NonCancellable) {
            saveToDatabase(data)   // キャンセルされても実行
            closeResources()       // キャンセルされても実行
        }
    }
}

// Compose内でのキャンセル例
@Composable
fun CancellableOperationScreen() {
    val scope = rememberCoroutineScope()
    var job by remember { mutableStateOf<Job?>(null) }
    var progress by remember { mutableFloatStateOf(0f) }

    Column(Modifier.padding(16.dp)) {
        LinearProgressIndicator(progress = { progress }, Modifier.fillMaxWidth())

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = {
                job = scope.launch {
                    for (i in 0..100) {
                        progress = i / 100f
                        delay(100)
                    }
                }
            }) { Text("開始") }

            OutlinedButton(onClick = { job?.cancel() }) { Text("キャンセル") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `job.cancel()` | Jobのキャンセル |
| `ensureActive()` | キャンセルチェック |
| `isActive` | アクティブ状態確認 |
| `NonCancellable` | キャンセル後の処理 |

- `delay()`等のsuspend関数はキャンセルに自動応答
- CPU処理中は`ensureActive()`で定期チェック
- `CancellationException`は必ず再スロー
- `withContext(NonCancellable)`でクリーンアップを保証

---

8種類のAndroidアプリテンプレート（Coroutine対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Scope](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-scope-2026)
- [Coroutine Dispatcher](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-dispatcher-2026)
- [Coroutine ExceptionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-handler-2026)
