---
title: "WorkManager完全ガイド — チェーン/並列/定期実行"
emoji: "⛓️"
type: "tech"
topics: ["android", "kotlin", "workmanager", "background"]
published: true
---

## この記事で学べること

**WorkManager**（Worker実装、チェーン実行、並列実行、定期実行、制約条件、進捗通知）を解説します。

---

## 基本Worker

```kotlin
class ImageCompressWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val imageUri = inputData.getString("image_uri")
            ?: return Result.failure()

        return try {
            val compressedUri = compressImage(imageUri)

            val output = workDataOf("compressed_uri" to compressedUri)
            Result.success(output)
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }

    private suspend fun compressImage(uri: String): String {
        // 画像圧縮処理
        return "compressed_$uri"
    }
}
```

---

## チェーン実行（順次）

```kotlin
// ダウンロード → 圧縮 → アップロード の順次実行
fun startImagePipeline(imageUrl: String) {
    val workManager = WorkManager.getInstance(context)

    val downloadWork = OneTimeWorkRequestBuilder<DownloadWorker>()
        .setInputData(workDataOf("url" to imageUrl))
        .build()

    val compressWork = OneTimeWorkRequestBuilder<CompressWorker>()
        .build()

    val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>()
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()

    workManager
        .beginWith(downloadWork)
        .then(compressWork)    // downloadの出力 → compressの入力
        .then(uploadWork)       // compressの出力 → uploadの入力
        .enqueue()
}
```

---

## 並列実行

```kotlin
// 複数画像を並列処理 → 結合
fun processMultipleImages(imageUrls: List<String>) {
    val workManager = WorkManager.getInstance(context)

    // 並列ワーク
    val parallelWorks = imageUrls.map { url ->
        OneTimeWorkRequestBuilder<CompressWorker>()
            .setInputData(workDataOf("url" to url))
            .addTag("compress")
            .build()
    }

    // 結合ワーク（全並列完了後に実行）
    val mergeWork = OneTimeWorkRequestBuilder<MergeWorker>()
        .setInputMerger(ArrayCreatingInputMerger::class.java)
        .build()

    workManager
        .beginWith(parallelWorks) // 並列実行
        .then(mergeWork)           // 全完了後にマージ
        .enqueue()
}
```

---

## 定期実行

```kotlin
// 1時間ごとにデータ同期
fun schedulePeriodic() {
    val syncWork = PeriodicWorkRequestBuilder<SyncWorker>(
        repeatInterval = 1, TimeUnit.HOURS,
        flexInterval = 15, TimeUnit.MINUTES // 実行ウィンドウ
    )
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .setRequiresBatteryNotLow(true)
                .build()
        )
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            WorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .build()

    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "data_sync",
        ExistingPeriodicWorkPolicy.KEEP, // 既存があれば維持
        syncWork
    )
}
```

---

## 進捗通知

```kotlin
class LongRunningWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val items = getItemsToProcess()

        items.forEachIndexed { index, item ->
            processItem(item)

            // 進捗更新
            setProgress(workDataOf("progress" to (index + 1) * 100 / items.size))
        }

        return Result.success()
    }

    override suspend fun getForegroundInfo(): ForegroundInfo {
        return ForegroundInfo(
            NOTIFICATION_ID,
            createNotification("処理中...")
        )
    }
}

// Compose側で進捗監視
@Composable
fun WorkProgressScreen() {
    val context = LocalContext.current
    val workManager = WorkManager.getInstance(context)

    val workInfo by workManager
        .getWorkInfosForUniqueWorkLiveData("my_work")
        .observeAsState()

    val progress = workInfo?.firstOrNull()
        ?.progress?.getInt("progress", 0) ?: 0
    val state = workInfo?.firstOrNull()?.state

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        LinearProgressIndicator(
            progress = { progress / 100f },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(Modifier.height(8.dp))

        Text("$progress% 完了")

        Text(
            when (state) {
                WorkInfo.State.RUNNING -> "実行中"
                WorkInfo.State.SUCCEEDED -> "完了"
                WorkInfo.State.FAILED -> "失敗"
                WorkInfo.State.ENQUEUED -> "待機中"
                else -> ""
            }
        )
    }
}
```

---

## Hilt統合

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: DataRepository
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            repository.syncAll()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

// Application
@HiltAndroidApp
class MyApp : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

---

## まとめ

| パターン | メソッド | 用途 |
|---------|---------|------|
| 順次実行 | `beginWith().then()` | パイプライン処理 |
| 並列実行 | `beginWith(listOf())` | 複数並列処理 |
| 定期実行 | `PeriodicWorkRequest` | 定期同期 |
| ユニーク | `enqueueUniqueWork` | 重複防止 |
| 進捗 | `setProgress()` | UI更新 |

- `CoroutineWorker`でsuspend関数対応
- `Constraints`でネットワーク・バッテリー条件指定
- `BackoffPolicy`で失敗時リトライ制御
- `@HiltWorker`でDI統合

---

8種類のAndroidアプリテンプレート（WorkManager活用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [バックグラウンドタスク選択](https://zenn.dev/myougatheaxo/articles/android-compose-background-task-2026)
- [ファイルダウンロード/アップロード](https://zenn.dev/myougatheaxo/articles/android-compose-file-download-upload-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
