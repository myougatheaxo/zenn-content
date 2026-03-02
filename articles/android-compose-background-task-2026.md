---
title: "バックグラウンドタスク選択ガイド — WorkManager/Service/AlarmManager比較"
emoji: "⚙️"
type: "tech"
topics: ["android", "kotlin", "workmanager", "background"]
published: true
---

## この記事で学べること

Androidの**バックグラウンドタスク**選択基準（WorkManager、Foreground Service、AlarmManager）を解説します。

---

## タスク種別と選択

```
即時実行 + UIフィードバック必要
  → Coroutine (viewModelScope)

即時実行 + 長時間 + ユーザーに見える
  → Foreground Service

遅延可能 + 確実に実行
  → WorkManager

正確な時刻に実行
  → AlarmManager

定期実行（15分以上間隔）
  → WorkManager PeriodicWork

定期実行（15分未満間隔）
  → AlarmManager + Foreground Service
```

---

## WorkManager（推奨）

```kotlin
// 即時実行（遅延可能タスク）
class SyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return try {
            repository.syncAll()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// スケジュール
val request = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .setRequiresBatteryNotLow(true)
        .build())
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
    .addTag("sync")
    .build()

WorkManager.getInstance(context)
    .enqueueUniqueWork("sync", ExistingWorkPolicy.KEEP, request)

// 定期実行
val periodicRequest = PeriodicWorkRequestBuilder<SyncWorker>(
    repeatInterval = 1, repeatIntervalTimeUnit = TimeUnit.HOURS
).setConstraints(Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .build()
).build()

WorkManager.getInstance(context)
    .enqueueUniquePeriodicWork("periodic_sync", ExistingPeriodicWorkPolicy.UPDATE, periodicRequest)
```

---

## Foreground Service

```kotlin
class UploadService : Service() {

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = createNotification()
        startForeground(NOTIFICATION_ID, notification)

        CoroutineScope(Dispatchers.IO).launch {
            try {
                val filePath = intent?.getStringExtra("file_path") ?: return@launch
                uploadFile(filePath) { progress ->
                    updateNotification(progress)
                }
            } finally {
                stopSelf()
            }
        }

        return START_NOT_STICKY
    }

    private fun createNotification(): Notification {
        val channel = NotificationChannel(
            "upload", "アップロード", NotificationManager.IMPORTANCE_LOW
        )
        getSystemService(NotificationManager::class.java).createNotificationChannel(channel)

        return NotificationCompat.Builder(this, "upload")
            .setSmallIcon(R.drawable.ic_upload)
            .setContentTitle("アップロード中...")
            .setProgress(100, 0, true)
            .setOngoing(true)
            .build()
    }

    private fun updateNotification(progress: Int) {
        val notification = NotificationCompat.Builder(this, "upload")
            .setSmallIcon(R.drawable.ic_upload)
            .setContentTitle("アップロード中...")
            .setProgress(100, progress, false)
            .setOngoing(true)
            .build()

        getSystemService(NotificationManager::class.java)
            .notify(NOTIFICATION_ID, notification)
    }

    override fun onBind(intent: Intent?) = null

    companion object {
        const val NOTIFICATION_ID = 1001
    }
}

// Compose から起動
Button(onClick = {
    val intent = Intent(context, UploadService::class.java).apply {
        putExtra("file_path", filePath)
    }
    ContextCompat.startForegroundService(context, intent)
}) {
    Text("アップロード開始")
}
```

---

## チェーンタスク（WorkManager）

```kotlin
// タスクA → タスクB → タスクC の順序実行
val downloadWork = OneTimeWorkRequestBuilder<DownloadWorker>().build()
val processWork = OneTimeWorkRequestBuilder<ProcessWorker>().build()
val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()

WorkManager.getInstance(context)
    .beginWith(downloadWork)
    .then(processWork)
    .then(uploadWork)
    .enqueue()

// 並列 → 合流
val taskA = OneTimeWorkRequestBuilder<TaskAWorker>().build()
val taskB = OneTimeWorkRequestBuilder<TaskBWorker>().build()
val mergeTask = OneTimeWorkRequestBuilder<MergeWorker>().build()

WorkManager.getInstance(context)
    .beginWith(listOf(taskA, taskB)) // A, B 並列
    .then(mergeTask) // 両方完了後に実行
    .enqueue()
```

---

## 比較表

```
| 特性 | WorkManager | FG Service | AlarmManager |
|------|-------------|------------|--------------|
| 確実な実行 | ✅ | ✅ | △ |
| 制約条件 | ✅ | ❌ | ❌ |
| 定期実行 | ✅(15min+) | ❌ | ✅ |
| 正確な時刻 | ❌ | ❌ | ✅ |
| チェーン | ✅ | ❌ | ❌ |
| 進捗通知 | △ | ✅ | ❌ |
| Doze対応 | ✅ | ✅ | △ |
| バッテリー | 良好 | 消費大 | 普通 |
```

---

## まとめ

- **WorkManager**が第一選択（ほとんどのケースで推奨）
- Foreground Serviceは長時間タスク+ユーザー可視性が必要な場合
- AlarmManagerは正確な時刻が必要な場合のみ
- `Constraints`でネットワーク/バッテリー条件指定
- `beginWith().then()`でタスクチェーン
- Android 14+で`FOREGROUND_SERVICE_TYPE`必須

---

8種類のAndroidアプリテンプレート（バックグラウンド処理設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-2026)
- [AlarmManager](https://zenn.dev/myougatheaxo/articles/android-compose-alarm-notification-2026)
- [Foreground Service](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
