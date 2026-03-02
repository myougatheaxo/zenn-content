---
title: "WorkManager Expedited完全ガイド — 優先実行/Foreground対応/即時タスク"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "workmanager"]
published: true
---

## この記事で学べること

**WorkManager Expedited**（即時実行ワーカー、setExpedited、ForegroundInfo、短期的な重要タスク）を解説します。

---

## Expedited Work

```kotlin
class UploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val fileUri = inputData.getString("file_uri") ?: return Result.failure()

        return try {
            // ファイルアップロード処理
            uploadFile(Uri.parse(fileUri))
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }

    override suspend fun getForegroundInfo(): ForegroundInfo {
        val notification = NotificationCompat.Builder(applicationContext, "upload")
            .setSmallIcon(R.drawable.ic_upload)
            .setContentTitle("アップロード中")
            .setContentText("ファイルをアップロードしています...")
            .setProgress(100, 0, true)
            .build()
        return ForegroundInfo(1001, notification)
    }

    private suspend fun uploadFile(uri: Uri) {
        // アップロード実装
        delay(5000) // シミュレーション
    }
}
```

---

## Expedited実行リクエスト

```kotlin
// Composable から実行
@Composable
fun UploadScreen() {
    val context = LocalContext.current

    Column(Modifier.padding(16.dp)) {
        Button(onClick = {
            val request = OneTimeWorkRequestBuilder<UploadWorker>()
                .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                .setInputData(workDataOf("file_uri" to "content://..."))
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED)
                        .build()
                )
                .setBackoffCriteria(
                    BackoffPolicy.EXPONENTIAL,
                    Duration.ofSeconds(30)
                )
                .build()

            WorkManager.getInstance(context).enqueueUniqueWork(
                "upload_file",
                ExistingWorkPolicy.KEEP,
                request
            )
        }) { Text("即時アップロード") }
    }
}
```

---

## 進捗監視

```kotlin
@Composable
fun WorkProgressScreen() {
    val context = LocalContext.current
    val workManager = WorkManager.getInstance(context)
    val workInfo by workManager
        .getWorkInfosForUniqueWorkLiveData("upload_file")
        .observeAsState()

    Column(Modifier.padding(16.dp)) {
        workInfo?.firstOrNull()?.let { info ->
            when (info.state) {
                WorkInfo.State.ENQUEUED -> Text("待機中...")
                WorkInfo.State.RUNNING -> {
                    CircularProgressIndicator()
                    Text("アップロード中")
                }
                WorkInfo.State.SUCCEEDED -> {
                    Icon(Icons.Default.CheckCircle, null, tint = Color(0xFF4CAF50))
                    Text("完了!")
                }
                WorkInfo.State.FAILED -> {
                    Icon(Icons.Default.Error, null, tint = Color.Red)
                    Text("失敗")
                }
                else -> Text("状態: ${info.state}")
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `setExpedited` | 優先実行リクエスト |
| `getForegroundInfo` | フォアグラウンド通知 |
| `OutOfQuotaPolicy` | クォータ超過時の挙動 |
| `setBackoffCriteria` | リトライ戦略 |

- `setExpedited`でシステムが優先的にワーカーを実行
- `getForegroundInfo`の実装が必須（Android 12未満で使用）
- `OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED`でフォールバック
- `enqueueUniqueWork`で重複実行を防止

---

8種類のAndroidアプリテンプレート（WorkManager対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager Basic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-basic-2026)
- [WorkManager Periodic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-periodic-2026)
- [Compose ForegroundService](https://zenn.dev/myougatheaxo/articles/android-compose-compose-foreground-service-2026)
