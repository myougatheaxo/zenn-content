---
title: "Firebase Performance Monitoring完全ガイド — カスタムトレース/ネットワーク/Compose"
emoji: "⚡"
type: "tech"
topics: ["android", "firebase", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Firebase Performance Monitoring**（自動計測、カスタムトレース、ネットワーク監視、Compose統合）を解説します。

---

## 初期設定

```kotlin
// build.gradle.kts (project)
plugins {
    id("com.google.firebase.firebase-perf") version "1.4.2" apply false
}

// build.gradle.kts (app)
plugins {
    id("com.google.firebase.firebase-perf")
}

dependencies {
    implementation(platform(libs.firebase.bom))
    implementation("com.google.firebase:firebase-perf-ktx")
}
```

---

## カスタムトレース

```kotlin
class DataSyncRepository @Inject constructor(
    private val api: SyncApi,
    private val dao: SyncDao
) {
    suspend fun syncAll(): Result<Int> {
        val trace = Firebase.performance.newTrace("data_sync")
        trace.start()

        return try {
            val items = api.fetchAll()
            trace.putAttribute("item_count", items.size.toString())
            trace.putMetric("api_items", items.size.toLong())

            dao.insertAll(items.map { it.toEntity() })
            trace.putMetric("db_inserts", items.size.toLong())

            Result.success(items.size)
        } catch (e: Exception) {
            trace.putAttribute("error", e.javaClass.simpleName)
            Result.failure(e)
        } finally {
            trace.stop()
        }
    }
}

// 拡張関数でシンプルに
suspend inline fun <T> measureTrace(name: String, block: (Trace) -> T): T {
    val trace = Firebase.performance.newTrace(name)
    trace.start()
    return try {
        block(trace)
    } finally {
        trace.stop()
    }
}

// 使用
suspend fun loadData() = measureTrace("load_data") { trace ->
    val data = repository.fetchData()
    trace.putMetric("count", data.size.toLong())
    data
}
```

---

## 画面描画パフォーマンス

```kotlin
@Composable
fun PerformanceTrackedScreen(
    screenName: String,
    content: @Composable () -> Unit
) {
    val trace = remember { Firebase.performance.newTrace("screen_$screenName") }

    DisposableEffect(screenName) {
        trace.start()
        trace.putAttribute("screen", screenName)
        onDispose { trace.stop() }
    }

    content()
}

// 使用
@Composable
fun HomeScreen() {
    PerformanceTrackedScreen("home") {
        // 画面コンテンツ
        Column { /* ... */ }
    }
}
```

---

## ネットワーク監視

```kotlin
// OkHttp は自動計測される
// カスタムHTTPメトリクスが必要な場合:

class ApiPerformanceInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val metric = Firebase.performance.newHttpMetric(
            request.url.toString(),
            FirebasePerformance.HttpMethod.GET
        )
        metric.start()

        return try {
            val response = chain.proceed(request)
            metric.setHttpResponseCode(response.code)
            metric.setResponseContentType(response.header("Content-Type"))
            metric.setResponsePayloadSize(response.body?.contentLength() ?: 0)
            response
        } finally {
            metric.stop()
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| カスタムトレース | `newTrace()` |
| HTTP監視 | `newHttpMetric()` |
| 属性追加 | `putAttribute()` |
| メトリクス | `putMetric()` |

- 自動計測でアプリ起動/画面描画/ネットワークを監視
- カスタムトレースでビジネスロジックのパフォーマンス計測
- Firebase ConsoleでリアルタイムにパフォーマンスダッシュボードB
- OkHttpは自動計測対応

---

8種類のAndroidアプリテンプレート（パフォーマンス監視対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Analytics](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-analytics-2026)
- [Macrobenchmark](https://zenn.dev/myougatheaxo/articles/android-compose-benchmark-macro-2026)
- [デバッグツール](https://zenn.dev/myougatheaxo/articles/android-compose-debug-tools-2026)
