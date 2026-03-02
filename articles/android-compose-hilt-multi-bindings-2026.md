---
title: "Hilt応用パターン完全ガイド — Multibindings/AssistedInject/Qualifier"
emoji: "💉"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt応用**（Multibindings、AssistedInject、Qualifier、カスタムScope、テスト差し替え）を解説します。

---

## Qualifier

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @IoDispatcher
    @Provides
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @DefaultDispatcher
    @Provides
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}

// 使用
class UserRepository @Inject constructor(
    private val api: UserApi,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) {
    suspend fun getUsers() = withContext(ioDispatcher) { api.getUsers() }
}
```

---

## Multibindings（Set/Map）

```kotlin
// インターフェース
interface Analytics {
    fun trackEvent(name: String, params: Map<String, String>)
}

// 実装
class FirebaseAnalyticsImpl @Inject constructor() : Analytics {
    override fun trackEvent(name: String, params: Map<String, String>) { /* Firebase */ }
}

class MixpanelAnalyticsImpl @Inject constructor() : Analytics {
    override fun trackEvent(name: String, params: Map<String, String>) { /* Mixpanel */ }
}

// Module
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {
    @Binds
    @IntoSet
    abstract fun bindFirebase(impl: FirebaseAnalyticsImpl): Analytics

    @Binds
    @IntoSet
    abstract fun bindMixpanel(impl: MixpanelAnalyticsImpl): Analytics
}

// 全実装を注入
class AnalyticsTracker @Inject constructor(
    private val analyticsSet: Set<@JvmSuppressWildcards Analytics>
) {
    fun trackEvent(name: String, params: Map<String, String> = emptyMap()) {
        analyticsSet.forEach { it.trackEvent(name, params) }
    }
}
```

---

## AssistedInject

```kotlin
class NotificationWorker @AssistedInject constructor(
    @Assisted private val appContext: Context,
    @Assisted private val params: WorkerParameters,
    private val notificationRepository: NotificationRepository
) : CoroutineWorker(appContext, params) {

    @AssistedFactory
    interface Factory {
        fun create(appContext: Context, params: WorkerParameters): NotificationWorker
    }

    override suspend fun doWork(): Result {
        notificationRepository.sendPendingNotifications()
        return Result.success()
    }
}

// HiltWorker（WorkManager統合）
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncRepository: SyncRepository
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        syncRepository.sync()
        return Result.success()
    }
}
```

---

## カスタムScope

```kotlin
@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ActivityRetainedScoped

@Module
@InstallIn(ActivityRetainedComponent::class)
abstract class ActivityScopedModule {
    @Binds
    @ActivityRetainedScoped
    abstract fun bindNavigator(impl: AppNavigatorImpl): AppNavigator
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `@Qualifier` | 同型の区別 |
| `@IntoSet` | 複数実装の集約 |
| `@AssistedInject` | 実行時引数+DI |
| `@Binds` | インターフェース→実装 |
| カスタムScope | ライフサイクル制御 |

- `@Qualifier`で同じ型の異なるインスタンスを区別
- Multibindingsで複数実装を自動収集
- `@AssistedInject`でDI引数と実行時引数を混在
- `@Binds`は`@Provides`より効率的

---

8種類のAndroidアプリテンプレート（Hilt DI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt DI基本](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
- [Hiltテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-hilt-2026)
- [マルチモジュール](https://zenn.dev/myougatheaxo/articles/android-compose-multi-module-2026)
