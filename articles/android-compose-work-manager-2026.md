---
title: "WorkManager + Compose連携ガイド — バックグラウンドタスク管理"
emoji: "⏰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "workmanager"]
published: true
---

## この記事で学べること

**WorkManager**でのバックグラウンドタスク管理（定期実行、チェーン、進捗通知）を解説します。

---

## セットアップ

```kotlin
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
            val userId = inputData.getString("user_id") ?: return Result.failure()
            // 同期処理
            syncData(userId)
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }

    private suspend fun syncData(userId: String) {
        // API呼び出し等
    }
}
```

---

## ワンタイム実行

```kotlin
fun enqueueSyncWork(context: Context, userId: String) {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .setRequiresBatteryNotLow(true)
        .build()

    val request = OneTimeWorkRequestBuilder<SyncWorker>()
        .setConstraints(constraints)
        .setInputData(workDataOf("user_id" to userId))
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            Duration.ofSeconds(30)
        )
        .addTag("sync")
        .build()

    WorkManager.getInstance(context)
        .enqueueUniqueWork("sync_$userId", ExistingWorkPolicy.REPLACE, request)
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
        .setInputData(workDataOf("user_id" to "current"))
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

## Composeで進捗監視

```kotlin
@Composable
fun SyncStatusScreen() {
    val context = LocalContext.current
    val workManager = remember { WorkManager.getInstance(context) }

    val workInfos by workManager
        .getWorkInfosByTagFlow("sync")
        .collectAsStateWithLifecycle(emptyList())

    LazyColumn {
        items(workInfos) { info ->
            WorkStatusItem(info)
        }
    }
}

@Composable
fun WorkStatusItem(info: WorkInfo) {
    val status = when (info.state) {
        WorkInfo.State.ENQUEUED -> "待機中"
        WorkInfo.State.RUNNING -> "実行中"
        WorkInfo.State.SUCCEEDED -> "完了"
        WorkInfo.State.FAILED -> "失敗"
        WorkInfo.State.BLOCKED -> "ブロック中"
        WorkInfo.State.CANCELLED -> "キャンセル"
    }

    ListItem(
        headlineContent = { Text("同期タスク") },
        supportingContent = { Text(status) },
        leadingContent = {
            when (info.state) {
                WorkInfo.State.RUNNING -> CircularProgressIndicator(Modifier.size(24.dp))
                WorkInfo.State.SUCCEEDED -> Icon(Icons.Default.Check, null, tint = Color.Green)
                WorkInfo.State.FAILED -> Icon(Icons.Default.Close, null, tint = Color.Red)
                else -> Icon(Icons.Default.Schedule, null)
            }
        }
    )
}
```

---

## まとめ

- `CoroutineWorker`で非同期バックグラウンド処理
- `Constraints`でネットワーク/バッテリー条件指定
- `PeriodicWorkRequest`で定期実行
- `enqueueUniqueWork`で重複防止
- `getWorkInfosByTagFlow`でCompose側から進捗監視
- `Result.retry()`+`BackoffPolicy`で自動リトライ

---

8種類のAndroidアプリテンプレート（WorkManager実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine例外処理](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-exception-2026)
- [Flow+Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
- [通知パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
