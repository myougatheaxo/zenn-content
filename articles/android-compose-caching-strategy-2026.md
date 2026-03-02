---
title: "キャッシュ戦略完全ガイド — メモリ/ディスク/HTTP/有効期限管理"
emoji: "💨"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "performance"]
published: true
---

## この記事で学べること

**キャッシュ戦略**（メモリキャッシュ、ディスクキャッシュ、HTTPキャッシュ、有効期限管理、キャッシュ無効化）を解説します。

---

## メモリキャッシュ（LRU）

```kotlin
class InMemoryCache<K, V>(maxSize: Int = 100) {
    private val cache = object : LruCache<K, V>(maxSize) {
        override fun sizeOf(key: K & Any, value: V & Any): Int = 1
    }

    fun get(key: K): V? = cache.get(key)
    fun put(key: K & Any, value: V & Any) { cache.put(key, value) }
    fun remove(key: K & Any) { cache.remove(key) }
    fun clear() { cache.evictAll() }
}

// Repository での使用
class UserRepository @Inject constructor(
    private val api: UserApi,
    private val cache: InMemoryCache<String, User>
) {
    suspend fun getUser(id: String): User {
        cache.get(id)?.let { return it }
        return api.getUser(id).also { cache.put(id, it) }
    }
}
```

---

## 有効期限付きキャッシュ

```kotlin
class TimedCache<K, V>(
    private val maxAge: Duration = 5.minutes
) {
    private data class Entry<V>(val value: V, val timestamp: Long)
    private val cache = ConcurrentHashMap<K, Entry<V>>()

    fun get(key: K): V? {
        val entry = cache[key] ?: return null
        return if (System.currentTimeMillis() - entry.timestamp < maxAge.inWholeMilliseconds) {
            entry.value
        } else {
            cache.remove(key)
            null
        }
    }

    fun put(key: K, value: V) {
        cache[key] = Entry(value, System.currentTimeMillis())
    }

    fun invalidate(key: K) { cache.remove(key) }
    fun invalidateAll() { cache.clear() }
}
```

---

## OkHttpキャッシュ

```kotlin
@Provides
@Singleton
fun provideOkHttpClient(@ApplicationContext context: Context): OkHttpClient {
    val cacheDir = File(context.cacheDir, "http_cache")
    val cache = Cache(cacheDir, 10L * 1024 * 1024) // 10MB

    return OkHttpClient.Builder()
        .cache(cache)
        .addInterceptor { chain ->
            val request = chain.request()
            if (hasNetwork(context)) {
                request.newBuilder()
                    .header("Cache-Control", "public, max-age=60") // 1分キャッシュ
                    .build()
            } else {
                request.newBuilder()
                    .header("Cache-Control", "public, only-if-cached, max-stale=86400") // オフライン24時間
                    .build()
            }.let { chain.proceed(it) }
        }
        .build()
}
```

---

## Repository パターン（キャッシュ統合）

```kotlin
class ArticleRepository @Inject constructor(
    private val api: ArticleApi,
    private val dao: ArticleDao,
    private val memoryCache: TimedCache<String, List<Article>>
) {
    fun getArticles(category: String): Flow<List<Article>> = flow {
        // 1. メモリキャッシュ
        memoryCache.get(category)?.let { emit(it); return@flow }

        // 2. ローカルDB
        val local = dao.getByCategory(category)
        if (local.isNotEmpty()) {
            emit(local.map { it.toDomain() })
            memoryCache.put(category, local.map { it.toDomain() })
        }

        // 3. API（バックグラウンド同期）
        try {
            val remote = api.getArticles(category)
            dao.insertAll(remote.map { it.toEntity() })
            val updated = remote.map { it.toDomain() }
            memoryCache.put(category, updated)
            emit(updated)
        } catch (e: IOException) {
            if (local.isEmpty()) throw e
        }
    }
}
```

---

## まとめ

| キャッシュ層 | 速度 | 永続性 |
|------------|------|--------|
| メモリ(LRU) | 最速 | プロセス死で消失 |
| ディスク(Room) | 速い | 永続 |
| HTTP(OkHttp) | ネットワーク | TTL管理 |

- 3層キャッシュで最適なパフォーマンス
- `TimedCache`で有効期限自動管理
- OkHttp CacheでHTTPレベルのキャッシュ
- オフライン時はローカルキャッシュにフォールバック

---

8種類のAndroidアプリテンプレート（キャッシュ戦略実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [Room CRUD](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-crud-2026)
