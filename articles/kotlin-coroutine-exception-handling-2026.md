---
title: "コルーチン例外処理パターン — CoroutineExceptionHandler/supervisorScope"
emoji: "⚠️"
type: "tech"
topics: ["android", "kotlin", "coroutine", "errorhandling"]
published: true
---

## この記事で学べること

コルーチンの**例外処理**（CoroutineExceptionHandler、supervisorScope、構造化並行性）を解説します。

---

## 基本の例外伝播

```kotlin
// launch: 例外は親に伝播 → アプリクラッシュ
viewModelScope.launch {
    throw RuntimeException("Crash!") // ❌ 未処理例外
}

// ✅ try-catchで捕捉
viewModelScope.launch {
    try {
        val data = repository.fetchData()
        _uiState.value = UiState.Success(data)
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message ?: "Unknown")
    }
}
```

---

## CoroutineExceptionHandler

```kotlin
// グローバルハンドラー（launch用、async非対応）
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("Coroutine", "Caught: ${exception.message}")
    _error.value = exception.message
}

viewModelScope.launch(handler) {
    repository.riskyOperation() // 例外がhandlerで捕捉される
}

// ⚠️ asyncでは効かない
viewModelScope.launch(handler) {
    val deferred = async {
        throw RuntimeException("async error")
    }
    deferred.await() // ← ここでCatch必要
}
```

---

## async の例外処理

```kotlin
// async例外は await() で投げられる
viewModelScope.launch {
    val deferred = async {
        repository.fetchData() // ここで例外発生
    }

    try {
        val data = deferred.await() // ← ここで捕捉
        _uiState.value = UiState.Success(data)
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message ?: "Error")
    }
}
```

---

## supervisorScope（部分失敗許容）

```kotlin
// 1つ失敗しても他は続行
suspend fun loadDashboard(): DashboardData = supervisorScope {
    val userDeferred = async {
        try { api.getUser() } catch (e: Exception) { null }
    }
    val postsDeferred = async {
        try { api.getPosts() } catch (e: Exception) { emptyList() }
    }
    val statsDeferred = async {
        try { api.getStats() } catch (e: Exception) { Stats.empty() }
    }

    DashboardData(
        user = userDeferred.await(),
        posts = postsDeferred.await(),
        stats = statsDeferred.await()
    )
}

// coroutineScopeとの違い
// coroutineScope: 1つ失敗→全キャンセル
// supervisorScope: 1つ失敗→他は継続
```

---

## CancellationException の扱い

```kotlin
// ❌ CancellationExceptionをcatchしてはいけない
viewModelScope.launch {
    try {
        longRunningTask()
    } catch (e: Exception) {
        // CancellationExceptionもcatchされてしまう
        // → キャンセルが効かなくなる
    }
}

// ✅ CancellationExceptionを再throwする
viewModelScope.launch {
    try {
        longRunningTask()
    } catch (e: CancellationException) {
        throw e // 再throw必須
    } catch (e: Exception) {
        _error.value = e.message
    }
}

// ✅ runCatching + ensureActive
viewModelScope.launch {
    val result = runCatching { repository.fetchData() }
    ensureActive() // キャンセル確認
    result.fold(
        onSuccess = { _uiState.value = UiState.Success(it) },
        onFailure = { _uiState.value = UiState.Error(it.message ?: "") }
    )
}
```

---

## retry パターン

```kotlin
suspend fun <T> retryWithBackoff(
    times: Int = 3,
    initialDelay: Long = 1000,
    factor: Double = 2.0,
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(times - 1) {
        try {
            return block()
        } catch (e: Exception) {
            if (e is CancellationException) throw e
        }
        delay(currentDelay)
        currentDelay = (currentDelay * factor).toLong()
    }
    return block() // 最後の試行は例外をそのまま投げる
}

// 使用
val data = retryWithBackoff(times = 3) {
    api.fetchData()
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `try-catch` | 基本の例外捕捉 |
| `CoroutineExceptionHandler` | launch用グローバルハンドラー |
| `supervisorScope` | 部分失敗許容 |
| `ensureActive()` | キャンセル確認 |
| `retryWithBackoff` | リトライ+指数バックオフ |

- `async`の例外は`await()`で捕捉
- `CancellationException`は必ず再throw
- `runCatching`使用後は`ensureActive()`確認
- ネットワークAPIは`retryWithBackoff`で堅牢化

---

8種類のAndroidアプリテンプレート（エラーハンドリング設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [コルーチン基礎](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-basics-2026)
- [Flow演算子ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-operators-2026)
- [Result型パターン](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-result-2026)
