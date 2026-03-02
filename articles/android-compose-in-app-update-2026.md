---
title: "アプリ内更新完全ガイド — In-App Updates/即時更新/柔軟更新/Compose対応"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "playstore"]
published: true
---

## この記事で学べること

**In-App Updates**（即時更新、柔軟更新、更新チェック、Compose UI連携）を解説します。

---

## 更新チェック

```kotlin
class AppUpdateHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val appUpdateManager = AppUpdateManagerFactory.create(context)

    fun checkForUpdate(): Flow<UpdateInfo> = callbackFlow {
        val task = appUpdateManager.appUpdateInfo
        task.addOnSuccessListener { info ->
            trySend(UpdateInfo(
                isAvailable = info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE,
                isImmediate = info.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE),
                isFlexible = info.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE),
                versionCode = info.availableVersionCode()
            ))
        }
        task.addOnFailureListener { trySend(UpdateInfo()) }
        awaitClose()
    }

    fun startImmediateUpdate(activity: Activity) {
        appUpdateManager.appUpdateInfo.addOnSuccessListener { info ->
            appUpdateManager.startUpdateFlowForResult(
                info, activity, AppUpdateOptions.newBuilder(AppUpdateType.IMMEDIATE).build(), 100
            )
        }
    }

    fun startFlexibleUpdate(activity: Activity): Flow<Int> = callbackFlow {
        val listener = InstallStateUpdatedListener { state ->
            when (state.installStatus()) {
                InstallStatus.DOWNLOADING -> {
                    val progress = ((state.bytesDownloaded() * 100) / state.totalBytesToDownload()).toInt()
                    trySend(progress)
                }
                InstallStatus.DOWNLOADED -> {
                    trySend(100)
                    close()
                }
                else -> {}
            }
        }
        appUpdateManager.registerListener(listener)

        appUpdateManager.appUpdateInfo.addOnSuccessListener { info ->
            appUpdateManager.startUpdateFlowForResult(
                info, activity, AppUpdateOptions.newBuilder(AppUpdateType.FLEXIBLE).build(), 101
            )
        }

        awaitClose { appUpdateManager.unregisterListener(listener) }
    }

    fun completeUpdate() {
        appUpdateManager.completeUpdate()
    }
}

data class UpdateInfo(
    val isAvailable: Boolean = false,
    val isImmediate: Boolean = false,
    val isFlexible: Boolean = false,
    val versionCode: Int = 0
)
```

---

## Compose UI

```kotlin
@Composable
fun UpdateBanner(viewModel: UpdateViewModel = hiltViewModel()) {
    val updateInfo by viewModel.updateInfo.collectAsStateWithLifecycle(UpdateInfo())
    val downloadProgress by viewModel.downloadProgress.collectAsStateWithLifecycle(0)
    val activity = LocalContext.current as Activity

    if (updateInfo.isAvailable) {
        Card(
            Modifier.fillMaxWidth().padding(16.dp),
            colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer)
        ) {
            Column(Modifier.padding(16.dp)) {
                Text("新しいバージョンがあります", style = MaterialTheme.typography.titleSmall)
                if (downloadProgress in 1..99) {
                    LinearProgressIndicator(
                        progress = { downloadProgress / 100f },
                        modifier = Modifier.fillMaxWidth().padding(vertical = 8.dp)
                    )
                }
                Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                    if (downloadProgress == 100) {
                        Button(onClick = { viewModel.completeUpdate() }) { Text("再起動して更新") }
                    } else {
                        Button(onClick = { viewModel.startFlexibleUpdate(activity) }) { Text("更新") }
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| 更新タイプ | 動作 |
|-----------|------|
| 即時 (IMMEDIATE) | 全画面、強制更新 |
| 柔軟 (FLEXIBLE) | バックグラウンドDL |

- `AppUpdateManager`で更新チェック
- 即時更新で重要なアップデートを強制
- 柔軟更新でバックグラウンドダウンロード
- `completeUpdate`でダウンロード完了後に適用

---

8種類のAndroidアプリテンプレート（更新機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アプリ内レビュー](https://zenn.dev/myougatheaxo/articles/android-compose-in-app-review-2026)
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/android-compose-google-play-release-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-2026)
