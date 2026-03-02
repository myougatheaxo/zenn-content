---
title: "Coroutine Mutex完全ガイド — 排他制御/クリティカルセクション/競合防止"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**Coroutine Mutex**（排他制御、withLock、競合状態の防止）を解説します。

---

## Mutex基本

```kotlin
class SafeCounter {
    private val mutex = Mutex()
    private var count = 0

    suspend fun increment() = mutex.withLock { count++ }
    suspend fun get() = mutex.withLock { count }
}

// 使用例
suspend fun main() = coroutineScope {
    val counter = SafeCounter()
    (1..1000).map {
        launch { counter.increment() }
    }.joinAll()
    println(counter.get()) // 必ず1000
}
```

---

## API呼び出しの排他

```kotlin
class TokenManager @Inject constructor(
    private val api: AuthApi,
    private val dataStore: DataStore<Preferences>
) {
    private val refreshMutex = Mutex()

    suspend fun getValidToken(): String {
        val token = getCurrentToken()
        if (token != null && !isExpired(token)) return token

        // 複数コルーチンからの同時リフレッシュを防止
        return refreshMutex.withLock {
            // 他のコルーチンが既にリフレッシュ済みかチェック
            val freshToken = getCurrentToken()
            if (freshToken != null && !isExpired(freshToken)) return@withLock freshToken

            val newToken = api.refreshToken().accessToken
            saveToken(newToken)
            newToken
        }
    }
}
```

---

## キャッシュの排他制御

```kotlin
class CachedRepository @Inject constructor(private val api: ItemApi) {
    private val cacheMutex = Mutex()
    private var cache: List<Item>? = null
    private var lastFetch: Long = 0

    suspend fun getItems(forceRefresh: Boolean = false): List<Item> {
        // キャッシュがあれば即返す（ロック不要）
        if (!forceRefresh && cache != null && System.currentTimeMillis() - lastFetch < 60_000) {
            return cache!!
        }

        return cacheMutex.withLock {
            // ダブルチェック
            if (!forceRefresh && cache != null && System.currentTimeMillis() - lastFetch < 60_000) {
                return@withLock cache!!
            }
            val items = api.getItems()
            cache = items
            lastFetch = System.currentTimeMillis()
            items
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Mutex()` | ミューテックス生成 |
| `withLock` | ロック付き実行 |
| `lock/unlock` | 手動ロック |
| `tryLock` | ノンブロッキング |

- `Mutex`でコルーチン間の排他制御
- `withLock`でクリティカルセクションを保護
- トークンリフレッシュの重複防止に最適
- `tryLock`でノンブロッキングなロック取得

---

8種類のAndroidアプリテンプレート（非同期処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [SupervisorJob](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-supervisor-2026)
- [Flow combine/zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-combine-2026)
- [Channel](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-channel-2026)
