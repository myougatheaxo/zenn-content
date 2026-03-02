---
title: "オフラインファースト設計 — Room + Retrofitでネットワーク不要のアプリを作る"
emoji: "📶"
type: "tech"
topics: ["android", "kotlin", "room", "architecture"]
published: true
---

## この記事で学べること

ネットワークがなくても動く**オフラインファーストアプリ**の設計パターンを解説します。Room（ローカルDB）を信頼源にし、Retrofitでサーバーと同期します。

---

## オフラインファーストとは

```
従来: UI → API → 表示（ネットワーク必須）
オフラインファースト: UI → Room → 表示（常に動く）
                              ↑
                     API → Room（バックグラウンド同期）
```

**Roomがデータの信頼源（Single Source of Truth）**。APIはRoomを更新するだけ。

---

## Repository パターン

```kotlin
class TaskRepository(
    private val dao: TaskDao,
    private val api: TaskApi
) {
    // UIはRoomのFlowを購読
    fun observeTasks(): Flow<List<Task>> = dao.getAllTasks()

    // リフレッシュ（API→Room）
    suspend fun refresh() {
        try {
            val remoteTasks = api.getTasks()
            dao.insertAll(remoteTasks.map { it.toEntity() })
        } catch (e: Exception) {
            // オフラインでもエラーにしない
        }
    }

    // 作成（Room→API）
    suspend fun createTask(task: Task) {
        val entity = task.toEntity(synced = false)
        dao.insert(entity)

        try {
            api.createTask(task.toRequest())
            dao.markSynced(entity.id)
        } catch (e: Exception) {
            // 後で同期
        }
    }
}
```

---

## DAO

```kotlin
@Dao
interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY createdAt DESC")
    fun getAllTasks(): Flow<List<TaskEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(tasks: List<TaskEntity>)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(task: TaskEntity)

    @Query("UPDATE tasks SET synced = 1 WHERE id = :id")
    suspend fun markSynced(id: String)

    @Query("SELECT * FROM tasks WHERE synced = 0")
    suspend fun getUnsyncedTasks(): List<TaskEntity>
}
```

---

## 同期ワーカー（WorkManager）

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters,
    private val repository: TaskRepository
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            repository.syncUnsyncedTasks()
            repository.refresh()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}
```

```kotlin
// 定期同期（15分ごと）
val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
    15, TimeUnit.MINUTES
).setConstraints(
    Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
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

## ネットワーク状態の監視

```kotlin
class NetworkMonitor(context: Context) {
    private val connectivityManager =
        context.getSystemService<ConnectivityManager>()

    val isOnline: Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                trySend(true)
            }
            override fun onLost(network: Network) {
                trySend(false)
            }
        }

        connectivityManager?.registerDefaultNetworkCallback(callback)

        awaitClose {
            connectivityManager?.unregisterNetworkCallback(callback)
        }
    }
}
```

```kotlin
@Composable
fun OfflineBanner(networkMonitor: NetworkMonitor) {
    val isOnline by networkMonitor.isOnline.collectAsStateWithLifecycle(true)

    AnimatedVisibility(visible = !isOnline) {
        Surface(
            color = MaterialTheme.colorScheme.errorContainer,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                "オフラインモード",
                modifier = Modifier.padding(8.dp),
                textAlign = TextAlign.Center
            )
        }
    }
}
```

---

## データフロー図

```
[UI] ← collectAsState ← [Room (Flow)]
                              ↑ insert
[WorkManager] → [API] → [Repository] → [Room]
    (定期同期)              ↑ 手動操作
                          [UI操作]
```

---

## まとめ

- **Room = Single Source of Truth**（UIはRoomだけを見る）
- APIはRoomを更新するバックグラウンド処理
- 未同期データに`synced`フラグを付けて後で同期
- WorkManagerで定期的にバックグラウンド同期
- NetworkMonitorでオフラインバナー表示

---

8種類のAndroidアプリテンプレート（ローカルDB設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Migration完全ガイド](https://zenn.dev/myougatheaxo/articles/room-migration-guide-2026)
- [WorkManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-workmanager-guide-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
