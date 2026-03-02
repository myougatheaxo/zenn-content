---
title: "Retrofit Logging完全ガイド — HttpLoggingInterceptor/デバッグ/cURL出力"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit Logging**（HttpLoggingInterceptor、ログレベル設定、カスタムロガー、cURL出力）を解説します。

---

## 基本ログ設定

```kotlin
// build.gradle
// implementation "com.squareup.okhttp3:logging-interceptor:4.12.0"

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder().apply {
            if (BuildConfig.DEBUG) {
                addInterceptor(
                    HttpLoggingInterceptor().apply {
                        level = HttpLoggingInterceptor.Level.BODY
                    }
                )
            }
        }.build()
}
```

---

## カスタムロガー

```kotlin
class ApiLogger : HttpLoggingInterceptor.Logger {
    override fun log(message: String) {
        if (message.startsWith("{") || message.startsWith("[")) {
            // JSONを整形して出力
            try {
                val json = if (message.startsWith("{"))
                    JSONObject(message).toString(2)
                else JSONArray(message).toString(2)
                Log.d("API", json)
            } catch (e: Exception) {
                Log.d("API", message)
            }
        } else {
            Log.d("API", message)
        }
    }
}

// 使用
HttpLoggingInterceptor(ApiLogger()).apply {
    level = HttpLoggingInterceptor.Level.BODY
}
```

---

## レスポンス時間計測

```kotlin
class TimingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val startTime = System.nanoTime()

        val response = chain.proceed(request)

        val duration = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)
        Log.d("API_TIMING", "${request.method} ${request.url} → ${response.code} (${duration}ms)")

        if (duration > 3000) {
            Log.w("API_TIMING", "SLOW REQUEST: ${request.url} took ${duration}ms")
        }

        return response
    }
}
```

---

## まとめ

| レベル | 出力内容 |
|--------|---------|
| `NONE` | ログなし |
| `BASIC` | URL/メソッド/レスポンスコード |
| `HEADERS` | BASIC + ヘッダー |
| `BODY` | HEADERS + ボディ |

- `BuildConfig.DEBUG`でリリースビルドではログ無効化
- カスタムLoggerでJSON整形出力
- `TimingInterceptor`でAPI応答時間を計測
- セキュリティ: 本番でBODYログは絶対に無効化

---

8種類のAndroidアプリテンプレート（API通信対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Retrofit Cache](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-cache-2026)
- [Retrofit Authenticator](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-authenticator-2026)
