---
title: "Kotlinコルーチン入門ガイド — launch/async/withContext"
emoji: "⚡"
type: "tech"
topics: ["android", "kotlin", "coroutine", "async"]
published: true
---

## この記事で学べること

Kotlinの**コルーチン**基礎（launch、async、withContext、Dispatcher）を解説します。

---

## launch — 結果不要の非同期処理

```kotlin
class UserViewModel : ViewModel() {
    fun saveUser(user: User) {
        viewModelScope.launch {
            repository.save(user)
            // 完了後の処理
        }
    }

    fun loadData() {
        viewModelScope.launch {
            _isLoading.value = true
            try {
                val users = repository.getAll()
                _users.value = users
            } catch (e: Exception) {
                _error.value = e.message
            } finally {
                _isLoading.value = false
            }
        }
    }
}
```

---

## async/await — 結果が必要な非同期処理

```kotlin
viewModelScope.launch {
    // 並行実行
    val userDeferred = async { repository.getUser(userId) }
    val postsDeferred = async { repository.getPosts(userId) }

    // 両方の結果を待つ
    val user = userDeferred.await()
    val posts = postsDeferred.await()

    _uiState.value = UiState(user = user, posts = posts)
}

// 複数APIの並行呼び出し
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val stats = async { api.getStats() }
    val notifications = async { api.getNotifications() }
    val profile = async { api.getProfile() }

    Dashboard(
        stats = stats.await(),
        notifications = notifications.await(),
        profile = profile.await()
    )
}
```

---

## withContext — スレッド切り替え

```kotlin
// Dispatchers:
// Main: UIスレッド（Compose更新）
// IO: ネットワーク/ディスクI/O
// Default: CPU重い計算

suspend fun processImage(bitmap: Bitmap): Bitmap {
    return withContext(Dispatchers.Default) {
        // CPU集約的な画像処理
        applyFilter(bitmap)
    }
}

suspend fun readFile(path: String): String {
    return withContext(Dispatchers.IO) {
        File(path).readText()
    }
}

// Repository内での使用
class UserRepository(private val dao: UserDao, private val api: UserApi) {
    suspend fun syncUsers() = withContext(Dispatchers.IO) {
        val remoteUsers = api.getUsers()
        dao.upsertAll(remoteUsers)
    }
}
```

---

## coroutineScope vs supervisorScope

```kotlin
// coroutineScope: 1つでも失敗したら全てキャンセル
suspend fun loadAllOrFail() = coroutineScope {
    val users = async { api.getUsers() }
    val posts = async { api.getPosts() } // これが失敗すると usersもキャンセル
    Pair(users.await(), posts.await())
}

// supervisorScope: 失敗しても他はキャンセルされない
suspend fun loadBestEffort() = supervisorScope {
    val users = async {
        try { api.getUsers() } catch (e: Exception) { emptyList() }
    }
    val posts = async {
        try { api.getPosts() } catch (e: Exception) { emptyList() }
    }
    Pair(users.await(), posts.await())
}
```

---

## delay / timeout

```kotlin
// delay: 一定時間待機
viewModelScope.launch {
    delay(3000) // 3秒待つ
    _showBanner.value = false
}

// withTimeout: タイムアウト設定
suspend fun fetchWithTimeout(): Result<Data> {
    return try {
        val data = withTimeout(5000) { // 5秒以内
            api.fetchData()
        }
        Result.success(data)
    } catch (e: TimeoutCancellationException) {
        Result.failure(Exception("タイムアウトしました"))
    }
}
```

---

## まとめ

| API | 用途 | 戻り値 |
|-----|------|--------|
| `launch` | Fire-and-forget | Job |
| `async` | 結果を返す並行処理 | Deferred<T> |
| `withContext` | スレッド切り替え | T |
| `coroutineScope` | 構造化並行性 | T |
| `supervisorScope` | 部分失敗許容 | T |
| `delay` | 非ブロッキング待機 | Unit |

- `viewModelScope`でViewModel内のコルーチン管理
- `Dispatchers.IO`でI/O処理、`Default`でCPU処理
- `withTimeout`でタイムアウト制御

---

8種類のAndroidアプリテンプレート（コルーチン設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [コルーチン例外処理](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-exception-2026)
- [Flow演算子ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-operators-2026)
- [Flow+Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
