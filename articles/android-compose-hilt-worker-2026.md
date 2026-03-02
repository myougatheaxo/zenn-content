---
title: "Hilt Worker完全ガイド — WorkManager DI/HiltWorker/AssistedInject"
emoji: "⚙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt Worker**（@HiltWorker、WorkManager DI、AssistedInject、定期実行）を解説します。

---

## @HiltWorker

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val repository: SyncRepository,
    private val notificationHelper: NotificationHelper
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return try {
            repository.syncAll()
            notificationHelper.showNotification("同期完了", "データの同期が完了しました")
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}
```

---

## WorkManager設定

```kotlin
// Application
@HiltAndroidApp
class App : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}

// 定期実行のスケジュール
@Module
@InstallIn(SingletonComponent::class)
object WorkModule {
    @Provides
    @Singleton
    fun provideWorkManager(@ApplicationContext context: Context): WorkManager =
        WorkManager.getInstance(context)
}

class SyncScheduler @Inject constructor(
    private val workManager: WorkManager
) {
    fun schedulePeriodic() {
        val request = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .setRequiresBatteryNotLow(true)
                    .build()
            )
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
            .build()

        workManager.enqueueUniquePeriodicWork(
            "sync_work", ExistingPeriodicWorkPolicy.KEEP, request
        )
    }
}
```

---

## Compose連携

```kotlin
@Composable
fun SyncStatusScreen() {
    val context = LocalContext.current
    val workManager = WorkManager.getInstance(context)
    val workInfo by workManager
        .getWorkInfosForUniqueWorkLiveData("sync_work")
        .observeAsState()

    Column(Modifier.padding(16.dp)) {
        val state = workInfo?.firstOrNull()?.state
        Text("同期ステータス: ${state?.name ?: "未スケジュール"}")

        Button(onClick = {
            val request = OneTimeWorkRequestBuilder<SyncWorker>().build()
            workManager.enqueue(request)
        }) {
            Text("今すぐ同期")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@HiltWorker` | Worker DI対応 |
| `@AssistedInject` | Context/Params注入 |
| `HiltWorkerFactory` | Worker生成 |
| `PeriodicWorkRequest` | 定期実行 |

- `@HiltWorker`でWorkerにDI注入
- `@AssistedInject`でContext/ParamsはWorkManager提供
- `HiltWorkerFactory`でDI対応Worker生成
- `PeriodicWorkRequest`で定期バックグラウンド処理

---

8種類のAndroidアプリテンプレート（バックグラウンド処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager Chain](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-chain-2026)
- [Hilt Testing](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
- [Hilt Multimodule](https://zenn.dev/myougatheaxo/articles/android-compose-dagger-hilt-multimodule-2026)
