---
title: "Firebase App Check完全ガイド — 不正アクセス防止/Play Integrity/デバッグ"
emoji: "🛡️"
type: "tech"
topics: ["android", "firebase", "kotlin", "security"]
published: true
---

## この記事で学べること

**Firebase App Check**（Play Integrity連携、カスタムプロバイダ、デバッグトークン、Retrofit統合）を解説します。

---

## 初期設定

```kotlin
// Application クラス
@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        FirebaseApp.initializeApp(this)

        val firebaseAppCheck = FirebaseAppCheck.getInstance()
        firebaseAppCheck.installAppCheckProviderFactory(
            if (BuildConfig.DEBUG) {
                DebugAppCheckProviderFactory.getInstance()
            } else {
                PlayIntegrityAppCheckProviderFactory.getInstance()
            }
        )
    }
}
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.appcheck.playintegrity)
    implementation(libs.firebase.appcheck.debug) // debugのみ
}
```

---

## トークン取得

```kotlin
class AppCheckRepository @Inject constructor() {
    suspend fun getAppCheckToken(): String {
        return suspendCoroutine { cont ->
            FirebaseAppCheck.getInstance()
                .getAppCheckToken(false)
                .addOnSuccessListener { result ->
                    cont.resume(result.token)
                }
                .addOnFailureListener { e ->
                    cont.resumeWithException(e)
                }
        }
    }
}
```

---

## OkHttp Interceptor

```kotlin
class AppCheckInterceptor @Inject constructor(
    private val appCheckRepository: AppCheckRepository
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = runBlocking { appCheckRepository.getAppCheckToken() }

        val request = chain.request().newBuilder()
            .addHeader("X-Firebase-AppCheck", token)
            .build()

        return chain.proceed(request)
    }
}

// OkHttp設定
@Provides
@Singleton
fun provideOkHttpClient(
    appCheckInterceptor: AppCheckInterceptor
): OkHttpClient {
    return OkHttpClient.Builder()
        .addInterceptor(appCheckInterceptor)
        .build()
}
```

---

## サーバー側検証（参考）

```kotlin
// Cloud Functions / サーバー側
// X-Firebase-AppCheck ヘッダーのトークンを検証
// Firebase Admin SDK: appCheck().verifyToken(token)

// Android側: 自前APIサーバーへの全リクエストに
// AppCheckトークンを自動付与
```

---

## デバッグ設定

```kotlin
// 1. デバッグビルドで起動 → Logcatにデバッグトークン表示
// D/Firebase: App Check debug token: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

// 2. Firebase Console → App Check → アプリ → デバッグトークン管理
//    → トークンを登録

// 3. CI/CDでの使用
// build.gradle.kts
android {
    buildTypes {
        debug {
            // CI環境変数からデバッグトークンを設定
            buildConfigField("String", "APP_CHECK_DEBUG_TOKEN",
                "\"${System.getenv("APP_CHECK_TOKEN") ?: ""}\"")
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 本番 | `PlayIntegrityAppCheckProviderFactory` |
| デバッグ | `DebugAppCheckProviderFactory` |
| API保護 | `AppCheckInterceptor` |
| トークン取得 | `getAppCheckToken()` |

- Play Integrityで正規アプリからのアクセスのみ許可
- OkHttp Interceptorで全APIリクエストにトークン付与
- デバッグトークンで開発時のテストをスムーズに
- Firebase Console/サーバーでトークン検証

---

8種類のAndroidアプリテンプレート（セキュリティ対策済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Analytics](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-analytics-2026)
- [Firebase Remote Config](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-remote-config-2026)
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
