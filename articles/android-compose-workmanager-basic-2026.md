---
title: "WorkManager基本ガイド — バックグラウンドタスク/OneTimeWork/制約条件"
emoji: "⚙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "workmanager"]
published: true
---

## この記事で学べること

**WorkManager基本**（OneTimeWorkRequest、Worker、制約条件、入出力データ）を解説します。

---

## 基本Worker

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val api = ApiClient.create()
            val localData = AppDatabase.getInstance(applicationContext).taskDao().getUnsynced()
            api.sync(localData)
            AppDatabase.getInstance(applicationContext).taskDao().markSynced(localData.map { it.id })
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// 実行
fun scheduleSyncWork(context: Context) {
    val request = OneTimeWorkRequestBuilder<SyncWorker>()
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
        .build()

    WorkManager.getInstance(context).enqueue(request)
}
```

---

## 入出力データ

```kotlin
class DownloadWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val url = inputData.getString("url") ?: return Result.failure()
        val fileName = inputData.getString("fileName") ?: "download"

        // ダウンロード処理
        val filePath = downloadFile(url, fileName)

        val output = workDataOf("filePath" to filePath, "fileSize" to File(filePath).length())
        return Result.success(output)
    }
}

// 実行と結果監視
fun downloadWithResult(context: Context, url: String) {
    val request = OneTimeWorkRequestBuilder<DownloadWorker>()
        .setInputData(workDataOf("url" to url, "fileName" to "data.json"))
        .build()

    WorkManager.getInstance(context).enqueue(request)
}
```

---

## Compose監視

```kotlin
@Composable
fun WorkStatusScreen() {
    val context = LocalContext.current
    val workManager = WorkManager.getInstance(context)

    val workInfo by workManager
        .getWorkInfosForUniqueWorkLiveData("sync")
        .observeAsState()

    val state = workInfo?.firstOrNull()?.state

    Column(Modifier.padding(16.dp)) {
        Text("同期状態: ${state?.name ?: "未実行"}")

        Button(onClick = {
            val request = OneTimeWorkRequestBuilder<SyncWorker>().build()
            workManager.enqueueUniqueWork("sync", ExistingWorkPolicy.REPLACE, request)
        }) {
            Text("同期開始")
        }

        if (state == WorkInfo.State.RUNNING) {
            CircularProgressIndicator(Modifier.padding(top = 16.dp))
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `OneTimeWorkRequest` | 一回限りのタスク |
| `CoroutineWorker` | suspend対応Worker |
| `Constraints` | 実行条件 |
| `workDataOf` | 入出力データ |

- `WorkManager`でアプリ終了後もタスク継続
- `Constraints`でネットワーク/充電状態を条件指定
- `BackoffPolicy`でリトライ間隔を制御
- `enqueueUniqueWork`で重複実行防止

---

8種類のAndroidアプリテンプレート（バックグラウンド処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager Periodic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-periodic-2026)
- [WorkManager Chain](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-chain-2026)
- [Hilt Worker](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-worker-2026)
