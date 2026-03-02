---
title: "WorkManager入門 — バックグラウンド処理を確実に実行する唯一の方法"
emoji: "⚙️"
type: "tech"
topics: ["android", "kotlin", "workmanager", "background"]
published: true
---

## この記事で学べること

「アプリを閉じてもデータ同期を続けたい」「毎日決まった時間にDBをクリーンアップしたい」。こういったバックグラウンド処理には**WorkManager**を使います。

---

## なぜWorkManagerなのか

| 手段 | 問題 |
|------|------|
| Thread/Coroutine | アプリ終了時にキャンセルされる |
| Service | Android 8以降、バックグラウンド実行に制限 |
| AlarmManager | 正確な時間指定向け、柔軟性が低い |
| **WorkManager** | アプリ終了後も実行、制約条件付き、チェーン可能 |

---

## 基本：OneTimeWorkRequest

```kotlin
// Worker定義
class CleanupWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // 30日以上前のデータを削除
        val dao = AppDatabase.getInstance(applicationContext).habitDao()
        dao.deleteOlderThan(System.currentTimeMillis() - 30L * 24 * 60 * 60 * 1000)
        return Result.success()
    }
}
```

```kotlin
// 実行
val request = OneTimeWorkRequestBuilder<CleanupWorker>()
    .build()

WorkManager.getInstance(context).enqueue(request)
```

---

## 定期実行：PeriodicWorkRequest

```kotlin
val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
    repeatInterval = 1, TimeUnit.HOURS  // 最低15分
).setConstraints(
    Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .setRequiresBatteryNotLow(true)
        .build()
).build()

WorkManager.getInstance(context)
    .enqueueUniquePeriodicWork(
        "sync",
        ExistingPeriodicWorkPolicy.KEEP,
        syncRequest
    )
```

---

## 制約条件

```kotlin
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)  // Wi-Fi or モバイル
    .setRequiresBatteryNotLow(true)                 // バッテリー残量あり
    .setRequiresCharging(false)                     // 充電中でなくてもOK
    .setRequiresStorageNotLow(true)                 // ストレージ余裕あり
    .build()
```

制約が満たされるまでWorkManagerが自動で待機してくれます。

---

## チェーン（直列・並列）

```kotlin
WorkManager.getInstance(context)
    .beginWith(listOf(downloadWork, parseWork))  // 並列
    .then(processWork)                           // 完了後に直列
    .then(uploadWork)                            // さらに直列
    .enqueue()
```

---

## 状態の監視

```kotlin
@Composable
fun WorkStatusScreen() {
    val context = LocalContext.current
    val workInfo = WorkManager.getInstance(context)
        .getWorkInfoByIdLiveData(workId)
        .observeAsState()

    when (workInfo.value?.state) {
        WorkInfo.State.RUNNING -> Text("実行中...")
        WorkInfo.State.SUCCEEDED -> Text("完了")
        WorkInfo.State.FAILED -> Text("失敗")
        else -> Text("待機中")
    }
}
```

---

## 依存関係

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.work:work-runtime-ktx:2.9.1")
}
```

---

## まとめ

- アプリ終了後も実行したい処理は**WorkManager**
- `OneTimeWorkRequest`で単発、`PeriodicWorkRequest`で定期
- `Constraints`でネットワーク・バッテリー条件を指定
- チェーンで複数タスクを直列・並列実行

---

8種類のAndroidアプリテンプレート（WorkManager統合も容易なMVVM設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin Coroutine入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-android-basics)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Android通知の実装ガイド](https://zenn.dev/myougatheaxo/articles/android-notifications-guide-2026)
