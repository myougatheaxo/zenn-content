---
title: "Coroutine Context完全ガイド — CoroutineContext/Job/Name/要素の合成"
emoji: "🧩"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Coroutine Context**（CoroutineContext、Job、CoroutineName、コンテキスト要素の合成）を解説します。

---

## CoroutineContext構成要素

```kotlin
// CoroutineContextは4つの要素の集合
val context: CoroutineContext =
    Dispatchers.IO +                      // Dispatcher
    SupervisorJob() +                     // Job
    CoroutineName("DataLoader") +         // Name（デバッグ用）
    CoroutineExceptionHandler { _, e ->   // ExceptionHandler
        Log.e("DataLoader", "Error", e)
    }

// 使用
val scope = CoroutineScope(context)
scope.launch {
    // Dispatchers.IO + SupervisorJob + Name + Handler
    val data = fetchData()
}
```

---

## コンテキストの継承と上書き

```kotlin
@HiltViewModel
class MyViewModel @Inject constructor() : ViewModel() {
    init {
        // viewModelScopeのContext: Main + SupervisorJob
        viewModelScope.launch {
            // 親のContextを継承

            withContext(Dispatchers.IO) {
                // Dispatcherだけ上書き（Jobは継承）
                fetchData()
            }

            launch(CoroutineName("SubTask")) {
                // 名前だけ追加（Dispatcher,Jobは継承）
                processData()
            }
        }
    }
}

// コンテキスト要素の取得
suspend fun logContext() {
    val name = coroutineContext[CoroutineName]?.name ?: "unnamed"
    val job = coroutineContext[Job]
    val dispatcher = coroutineContext[ContinuationInterceptor]
    println("Name: $name, Job: $job, Dispatcher: $dispatcher")
}
```

---

## カスタムContext要素

```kotlin
// カスタムコンテキスト要素（ユーザーID伝搬など）
class UserId(val id: String) : AbstractCoroutineContextElement(UserId) {
    companion object Key : CoroutineContext.Key<UserId>
}

// 使用
val scope = CoroutineScope(Dispatchers.Main + UserId("user123"))

scope.launch {
    val userId = coroutineContext[UserId]?.id
    println("User: $userId") // "user123"

    launch {
        // 子コルーチンにも継承される
        val childUserId = coroutineContext[UserId]?.id
        println("Child User: $childUserId") // "user123"
    }
}
```

---

## まとめ

| 要素 | 用途 |
|------|------|
| `Dispatcher` | 実行スレッド |
| `Job` | ライフサイクル管理 |
| `CoroutineName` | デバッグ識別 |
| `ExceptionHandler` | 例外処理 |

- `+`演算子でコンテキスト要素を合成
- 子コルーチンは親のContextを継承
- `withContext()`で一部要素を上書き
- カスタム要素でリクエストスコープの情報を伝搬

---

8種類のAndroidアプリテンプレート（Coroutine対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Dispatcher](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-dispatcher-2026)
- [Coroutine Scope](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-scope-2026)
- [Coroutine ExceptionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-handler-2026)
