---
title: "WorkManagerチェーン完全ガイド — 順次実行/並列実行/ユニーク/定期"
emoji: "⛓️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "workmanager"]
published: true
---

## この記事で学べること

**WorkManagerチェーン**（順次実行、並列実行、ユニークワーク、定期実行）を解説します。

---

## ワークチェーン

```kotlin
// 順次実行: ダウンロード → 加工 → アップロード
val chain = WorkManager.getInstance(context)
    .beginWith(OneTimeWorkRequestBuilder<DownloadWorker>()
        .setInputData(workDataOf("url" to imageUrl))
        .build())
    .then(OneTimeWorkRequestBuilder<ProcessWorker>().build())
    .then(OneTimeWorkRequestBuilder<UploadWorker>().build())
    .enqueue()

// 並列 → 合流
val download1 = OneTimeWorkRequestBuilder<DownloadWorker>()
    .setInputData(workDataOf("url" to url1)).build()
val download2 = OneTimeWorkRequestBuilder<DownloadWorker>()
    .setInputData(workDataOf("url" to url2)).build()

WorkManager.getInstance(context)
    .beginWith(listOf(download1, download2))
    .then(OneTimeWorkRequestBuilder<MergeWorker>().build())
    .enqueue()
```

---

## ユニークワーク

```kotlin
// ユニーク一回限り
WorkManager.getInstance(context).enqueueUniqueWork(
    "sync_data",
    ExistingWorkPolicy.KEEP, // REPLACE, APPEND, APPEND_OR_REPLACE
    OneTimeWorkRequestBuilder<SyncWorker>()
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()
)

// 定期実行
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "periodic_sync",
    ExistingPeriodicWorkPolicy.UPDATE,
    PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .setRequiresBatteryNotLow(true)
                .build()
        )
        .build()
)
```

---

## Compose進捗表示

```kotlin
@Composable
fun WorkProgressScreen() {
    val context = LocalContext.current
    val workManager = WorkManager.getInstance(context)
    val workInfo by workManager
        .getWorkInfosForUniqueWorkLiveData("sync_data")
        .observeAsState()

    Column(Modifier.padding(16.dp)) {
        workInfo?.firstOrNull()?.let { info ->
            when (info.state) {
                WorkInfo.State.RUNNING -> {
                    val progress = info.progress.getInt("progress", 0)
                    LinearProgressIndicator(
                        progress = { progress / 100f },
                        modifier = Modifier.fillMaxWidth()
                    )
                    Text("同期中: $progress%")
                }
                WorkInfo.State.SUCCEEDED -> Text("同期完了")
                WorkInfo.State.FAILED -> Text("同期失敗", color = MaterialTheme.colorScheme.error)
                WorkInfo.State.ENQUEUED -> Text("待機中...")
                else -> {}
            }
        }

        Button(onClick = {
            workManager.enqueueUniqueWork(
                "sync_data",
                ExistingWorkPolicy.REPLACE,
                OneTimeWorkRequestBuilder<SyncWorker>().build()
            )
        }) { Text("同期開始") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `beginWith...then` | 順次チェーン |
| `beginWith(list)` | 並列実行 |
| `enqueueUniqueWork` | ユニークワーク |
| `PeriodicWorkRequest` | 定期実行 |

- `then()`で順次チェーン実行
- `beginWith(list)`で並列実行→合流
- `enqueueUniqueWork`で重複実行防止
- `getWorkInfosForUniqueWorkLiveData`で進捗監視

---

8種類のAndroidアプリテンプレート（バックグラウンド処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [Foreground Service](https://zenn.dev/myougatheaxo/articles/android-compose-foreground-service-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-2026)
