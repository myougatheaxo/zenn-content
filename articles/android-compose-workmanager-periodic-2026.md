---
title: "WorkManager Periodic完全ガイド — 定期実行/最小間隔/FlexInterval"
emoji: "🔁"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "workmanager"]
published: true
---

## この記事で学べること

**WorkManager Periodic**（PeriodicWorkRequest、定期実行、FlexInterval、定期タスク管理）を解説します。

---

## 基本定期実行

```kotlin
class DailyCleanupWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val db = AppDatabase.getInstance(applicationContext)
        val cutoff = System.currentTimeMillis() - 30L * 24 * 60 * 60 * 1000 // 30日前
        db.logDao().deleteOlderThan(cutoff)
        return Result.success()
    }
}

fun scheduleDailyCleanup(context: Context) {
    val request = PeriodicWorkRequestBuilder<DailyCleanupWorker>(
        repeatInterval = 24, TimeUnit.HOURS
    )
        .setConstraints(
            Constraints.Builder()
                .setRequiresCharging(true)
                .build()
        )
        .build()

    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "daily_cleanup",
        ExistingPeriodicWorkPolicy.KEEP,
        request
    )
}
```

---

## FlexInterval

```kotlin
fun scheduleFlexWork(context: Context) {
    // 1時間ごとに実行、ただし最後の15分間のウィンドウ内で実行
    val request = PeriodicWorkRequestBuilder<SyncWorker>(
        repeatInterval = 1, TimeUnit.HOURS,
        flexTimeInterval = 15, TimeUnit.MINUTES
    )
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()

    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "periodic_sync",
        ExistingPeriodicWorkPolicy.UPDATE,
        request
    )
}
```

---

## 状態管理UI

```kotlin
@Composable
fun PeriodicWorkManager() {
    val context = LocalContext.current
    val workManager = WorkManager.getInstance(context)

    val syncWork by workManager
        .getWorkInfosForUniqueWorkLiveData("periodic_sync")
        .observeAsState(emptyList())

    val isScheduled = syncWork.any { !it.state.isFinished }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text("自動同期", style = MaterialTheme.typography.titleMedium)
            Switch(
                checked = isScheduled,
                onCheckedChange = { enabled ->
                    if (enabled) {
                        scheduleFlexWork(context)
                    } else {
                        workManager.cancelUniqueWork("periodic_sync")
                    }
                }
            )
        }

        if (isScheduled) {
            Text("状態: ${syncWork.firstOrNull()?.state?.name}", style = MaterialTheme.typography.bodySmall)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `PeriodicWorkRequest` | 定期実行タスク |
| `repeatInterval` | 繰り返し間隔（最小15分） |
| `flexTimeInterval` | 実行ウィンドウ |
| `enqueueUniquePeriodicWork` | 重複防止 |

- 最小繰り返し間隔は15分
- `FlexInterval`で実行タイミングのウィンドウを指定
- `ExistingPeriodicWorkPolicy.KEEP`で既存タスク維持
- `cancelUniqueWork`でキャンセル

---

8種類のAndroidアプリテンプレート（バックグラウンド処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager Basic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-basic-2026)
- [WorkManager Chain](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-chain-2026)
- [Hilt Worker](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-worker-2026)
