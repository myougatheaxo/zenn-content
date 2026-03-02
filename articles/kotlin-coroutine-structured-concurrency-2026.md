---
title: "構造化並行性ガイド — coroutineScope/Job/キャンセル伝播"
emoji: "🌊"
type: "tech"
topics: ["android", "kotlin", "coroutine", "concurrency"]
published: true
---

## この記事で学べること

Kotlinの**構造化並行性**（coroutineScope、Job階層、キャンセル伝播、並行パターン）を解説します。

---

## 構造化並行性とは

```kotlin
// 親コルーチンは子の完了を待つ
viewModelScope.launch { // 親
    launch { task1() } // 子1
    launch { task2() } // 子2
    // 子1と子2の両方が完了するまで待機
}

// 親がキャンセルされると子も全てキャンセル
// 子が失敗すると親もキャンセル → 他の子もキャンセル
```

---

## coroutineScope（全成功 or 全失敗）

```kotlin
// 全ての子コルーチンが成功しないと例外
suspend fun loadAllData() = coroutineScope {
    val users = async { api.getUsers() }
    val posts = async { api.getPosts() }
    // 1つでも失敗 → 全てキャンセル → 例外throw

    DashboardData(users.await(), posts.await())
}

// ViewModel から呼ぶ
viewModelScope.launch {
    try {
        val data = loadAllData()
        _uiState.value = UiState.Success(data)
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message ?: "Error")
    }
}
```

---

## supervisorScope（部分失敗OK）

```kotlin
suspend fun loadBestEffort() = supervisorScope {
    val users = async {
        try { api.getUsers() } catch (e: Exception) { emptyList() }
    }
    val posts = async {
        try { api.getPosts() } catch (e: Exception) { emptyList() }
    }
    val stats = async {
        try { api.getStats() } catch (e: Exception) { null }
    }

    // 1つが失敗しても他は続行
    DashboardData(users.await(), posts.await(), stats.await())
}
```

---

## Job 階層

```kotlin
val parentJob = viewModelScope.launch {
    val childJob1 = launch {
        repeat(1000) {
            delay(100)
            // 処理
        }
    }

    val childJob2 = launch {
        repeat(1000) {
            delay(100)
            // 処理
        }
    }

    // 特定の子だけキャンセル
    delay(5000)
    childJob1.cancel() // childJob2は継続

    // 子の完了を待つ
    childJob2.join()
}

// 全体キャンセル
// parentJob.cancel() → childJob1, childJob2 両方キャンセル
```

---

## キャンセル協力

```kotlin
// ❌ キャンセルに応じない
suspend fun badTask() {
    var i = 0
    while (i < 1000000) {
        // 重い計算
        i++
    }
}

// ✅ キャンセルに応じる
suspend fun goodTask() {
    var i = 0
    while (i < 1000000) {
        ensureActive() // キャンセルチェック
        // 重い計算
        i++
    }
}

// ✅ yield() で他のコルーチンにも実行機会を与える
suspend fun cooperativeTask() {
    for (item in largeList) {
        yield() // 協力的な中断点
        process(item)
    }
}

// ✅ isActive チェック
suspend fun taskWithCleanup() = coroutineScope {
    launch {
        try {
            while (isActive) {
                doWork()
                delay(1000)
            }
        } finally {
            // クリーンアップ（キャンセル時も実行）
            withContext(NonCancellable) {
                saveProgress()
            }
        }
    }
}
```

---

## 並行パターン

```kotlin
// Fan-out: 1つのチャネルから複数ワーカーが消費
suspend fun fanOut() = coroutineScope {
    val channel = Channel<Int>(Channel.BUFFERED)

    // 複数ワーカー
    repeat(3) { workerId ->
        launch {
            for (item in channel) {
                println("Worker $workerId processing $item")
                processItem(item)
            }
        }
    }

    // プロデューサー
    launch {
        for (i in 1..100) {
            channel.send(i)
        }
        channel.close()
    }
}

// 並列数制限
suspend fun processWithLimit(items: List<Item>, concurrency: Int = 5) = coroutineScope {
    val semaphore = Semaphore(concurrency)

    items.map { item ->
        async {
            semaphore.withPermit {
                processItem(item)
            }
        }
    }.awaitAll()
}
```

---

## まとめ

| パターン | 失敗時 | 用途 |
|---------|--------|------|
| `coroutineScope` | 全キャンセル | 全成功必須の処理 |
| `supervisorScope` | 部分続行 | ベストエフォート |
| `viewModelScope` | ViewModel破棄で全キャンセル | UI関連処理 |

- 親のキャンセル→全子キャンセル（自動）
- `ensureActive()`/`yield()`でキャンセル協力
- `NonCancellable`でキャンセル時のクリーンアップ
- `Semaphore`で並行数制限

---

8種類のAndroidアプリテンプレート（コルーチン設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [コルーチン基礎](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-basics-2026)
- [コルーチン例外処理](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-exception-handling-2026)
- [Flow/Channel](https://zenn.dev/myougatheaxo/articles/kotlin-flow-channel-2026)
