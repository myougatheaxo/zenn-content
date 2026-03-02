---
title: "SupervisorJob完全ガイド — SupervisorJob/supervisorScope/子Job独立"
emoji: "👔"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**SupervisorJob**（SupervisorJob、supervisorScope、子Job独立失敗、構造化並行性）を解説します。

---

## SupervisorJobの基本

```kotlin
// 通常のJob: 1つの子が失敗 → 全子がキャンセル
// SupervisorJob: 1つの子が失敗 → 他の子は継続

class DataSyncService @Inject constructor(
    private val userRepo: UserRepository,
    private val settingsRepo: SettingsRepository,
    private val analyticsRepo: AnalyticsRepository
) {
    suspend fun syncAll(): SyncResult = supervisorScope {
        val userResult = async { userRepo.sync() }
        val settingsResult = async { settingsRepo.sync() }
        val analyticsResult = async { analyticsRepo.sync() }

        SyncResult(
            userSuccess = runCatching { userResult.await() }.isSuccess,
            settingsSuccess = runCatching { settingsResult.await() }.isSuccess,
            analyticsSuccess = runCatching { analyticsResult.await() }.isSuccess
        )
    }
}

data class SyncResult(
    val userSuccess: Boolean,
    val settingsSuccess: Boolean,
    val analyticsSuccess: Boolean
)
```

---

## ViewModelでのSupervisorJob

```kotlin
@HiltViewModel
class SyncViewModel @Inject constructor(
    private val syncService: DataSyncService
) : ViewModel() {
    // viewModelScopeは内部でSupervisorJobを使用
    private val _syncState = MutableStateFlow<SyncState>(SyncState.Idle)
    val syncState = _syncState.asStateFlow()

    fun syncAll() {
        viewModelScope.launch {
            _syncState.value = SyncState.Loading
            val result = syncService.syncAll()
            _syncState.value = if (result.userSuccess && result.settingsSuccess)
                SyncState.Success(result)
            else SyncState.PartialSuccess(result)
        }
    }
}

sealed class SyncState {
    data object Idle : SyncState()
    data object Loading : SyncState()
    data class Success(val result: SyncResult) : SyncState()
    data class PartialSuccess(val result: SyncResult) : SyncState()
}
```

---

## 並列ダウンロード

```kotlin
suspend fun downloadImages(urls: List<String>): List<Result<ByteArray>> =
    supervisorScope {
        urls.map { url ->
            async {
                runCatching { api.downloadImage(url) }
            }
        }.awaitAll()
    }
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SupervisorJob` | 子Job独立 |
| `supervisorScope` | スコープ内で独立 |
| `runCatching` | 個別エラー捕捉 |
| `viewModelScope` | 内部でSupervisorJob |

- `supervisorScope`で子Jobの失敗が他に伝播しない
- `runCatching`で個別の結果をResult型で取得
- `viewModelScope`は内部でSupervisorJobを使用
- 並列処理で部分的失敗を許容する場合に最適

---

8種類のAndroidアプリテンプレート（非同期処理実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Exception](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-2026)
- [Coroutine Channel](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-channel-2026)
- [Flow combine/zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-combine-2026)
