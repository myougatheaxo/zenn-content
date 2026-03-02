---
title: "WorkManager完全ガイド — バックグラウンドタスクの定期実行"
emoji: "🔧"
type: "tech"
topics: ["android", "kotlin", "workmanager", "background"]
published: true
---

## この記事で学べること

Android**WorkManager**を使ったバックグラウンドタスクの実装方法を解説します。

---

## 依存関係

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.work:work-runtime-ktx:2.10.0")
}
```

---

## 基本のWorker

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val repository = SyncRepository(applicationContext)
            repository.syncData()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
}
```

---

## ワンショット実行

```kotlin
fun scheduleOneTimeSync(context: Context) {
    val request = OneTimeWorkRequestBuilder<SyncWorker>()
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .setRequiresBatteryNotLow(true)
                .build()
        )
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            Duration.ofMinutes(1)
        )
        .addTag("sync")
        .build()

    WorkManager.getInstance(context)
        .enqueueUniqueWork(
            "one_time_sync",
            ExistingWorkPolicy.REPLACE,
            request
        )
}
```

---

## 定期実行

```kotlin
fun schedulePeriodicSync(context: Context) {
    val request = PeriodicWorkRequestBuilder<SyncWorker>(
        repeatInterval = 1, TimeUnit.HOURS,
        flexInterval = 15, TimeUnit.MINUTES
    )
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .addTag("periodic_sync")
        .build()

    WorkManager.getInstance(context)
        .enqueueUniquePeriodicWork(
            "periodic_sync",
            ExistingPeriodicWorkPolicy.KEEP,
            request
        )
}
```

---

## データの受け渡し

```kotlin
class UploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // 入力データ取得
        val filePath = inputData.getString("file_path") ?: return Result.failure()
        val userId = inputData.getString("user_id") ?: return Result.failure()

        val url = uploadFile(filePath, userId)

        // 結果データ返却
        return Result.success(
            workDataOf("upload_url" to url)
        )
    }
}

// データ付きリクエスト
fun uploadFile(context: Context, filePath: String, userId: String) {
    val request = OneTimeWorkRequestBuilder<UploadWorker>()
        .setInputData(
            workDataOf(
                "file_path" to filePath,
                "user_id" to userId
            )
        )
        .build()

    WorkManager.getInstance(context).enqueue(request)
}
```

---

## チェーン実行

```kotlin
fun processAndUpload(context: Context) {
    val compressWork = OneTimeWorkRequestBuilder<CompressWorker>().build()
    val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()
    val notifyWork = OneTimeWorkRequestBuilder<NotifyWorker>().build()

    WorkManager.getInstance(context)
        .beginWith(compressWork)    // 1. 圧縮
        .then(uploadWork)           // 2. アップロード
        .then(notifyWork)           // 3. 通知
        .enqueue()

    // 並列→直列
    val download1 = OneTimeWorkRequestBuilder<DownloadWorker>()
        .setInputData(workDataOf("url" to "url1"))
        .build()
    val download2 = OneTimeWorkRequestBuilder<DownloadWorker>()
        .setInputData(workDataOf("url" to "url2"))
        .build()
    val merge = OneTimeWorkRequestBuilder<MergeWorker>().build()

    WorkManager.getInstance(context)
        .beginWith(listOf(download1, download2)) // 並列ダウンロード
        .then(merge)                              // マージ
        .enqueue()
}
```

---

## 進捗の監視（Compose）

```kotlin
@Composable
fun WorkStatusScreen() {
    val context = LocalContext.current
    val workManager = remember { WorkManager.getInstance(context) }

    val workInfo by workManager
        .getWorkInfosForUniqueWorkLiveData("one_time_sync")
        .observeAsState()

    Column(Modifier.padding(16.dp)) {
        val info = workInfo?.firstOrNull()
        when (info?.state) {
            WorkInfo.State.ENQUEUED -> Text("待機中...")
            WorkInfo.State.RUNNING -> {
                CircularProgressIndicator()
                Text("同期中...")
            }
            WorkInfo.State.SUCCEEDED -> {
                val url = info.outputData.getString("upload_url")
                Text("完了: $url")
            }
            WorkInfo.State.FAILED -> Text("失敗")
            WorkInfo.State.CANCELLED -> Text("キャンセル済み")
            else -> Text("未スケジュール")
        }

        Button(onClick = { scheduleOneTimeSync(context) }) {
            Text("同期開始")
        }
    }
}
```

---

## まとめ

- `CoroutineWorker`でsuspend対応のWorker作成
- `Constraints`でネットワーク/バッテリー条件設定
- `OneTimeWorkRequest`で1回、`PeriodicWorkRequest`で定期実行
- `workDataOf`でデータ入出力
- `beginWith().then()`でチェーン実行
- `enqueueUniqueWork`で重複防止
- Composeで`observeAsState()`による進捗表示

---

8種類のAndroidアプリテンプレート（WorkManager統合可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AlarmManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-alarm-scheduler-2026)
- [Foreground Serviceガイド](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
- [Coroutines & Flowガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
