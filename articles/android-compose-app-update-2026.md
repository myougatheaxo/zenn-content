---
title: "アプリ内更新完全ガイド — In-App Update API/即時/柔軟更新"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "playstore"]
published: true
---

## この記事で学べること

**In-App Update**（即時更新、柔軟更新、更新チェック、Compose UI連携、テスト方法）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.google.android.play:app-update:2.1.0")
    implementation("com.google.android.play:app-update-ktx:2.1.0")
}
```

---

## AppUpdateManager

```kotlin
class AppUpdateHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val appUpdateManager = AppUpdateManagerFactory.create(context)

    sealed interface UpdateState {
        data object NotAvailable : UpdateState
        data class Available(val isImmediate: Boolean) : UpdateState
        data class Downloading(val progress: Float) : UpdateState
        data object Downloaded : UpdateState
        data class Failed(val message: String) : UpdateState
    }

    private val _updateState = MutableStateFlow<UpdateState>(UpdateState.NotAvailable)
    val updateState: StateFlow<UpdateState> = _updateState

    fun checkForUpdate() {
        appUpdateManager.appUpdateInfo.addOnSuccessListener { info ->
            when {
                info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE &&
                    info.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE) -> {
                    _updateState.value = UpdateState.Available(isImmediate = true)
                }
                info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE &&
                    info.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE) -> {
                    _updateState.value = UpdateState.Available(isImmediate = false)
                }
                info.installStatus() == InstallStatus.DOWNLOADED -> {
                    _updateState.value = UpdateState.Downloaded
                }
            }
        }
    }

    fun startUpdate(activity: Activity, type: Int) {
        appUpdateManager.appUpdateInfo.addOnSuccessListener { info ->
            appUpdateManager.startUpdateFlowForResult(info, activity, AppUpdateOptions.newBuilder(type).build(), 100)
        }

        if (type == AppUpdateType.FLEXIBLE) {
            appUpdateManager.registerListener { state ->
                when (state.installStatus()) {
                    InstallStatus.DOWNLOADING -> {
                        val progress = state.bytesDownloaded().toFloat() / state.totalBytesToDownload()
                        _updateState.value = UpdateState.Downloading(progress)
                    }
                    InstallStatus.DOWNLOADED -> {
                        _updateState.value = UpdateState.Downloaded
                    }
                    InstallStatus.FAILED -> {
                        _updateState.value = UpdateState.Failed("更新に失敗しました")
                    }
                    else -> {}
                }
            }
        }
    }

    fun completeUpdate() {
        appUpdateManager.completeUpdate()
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun UpdateBanner(updateHelper: AppUpdateHelper) {
    val updateState by updateHelper.updateState.collectAsStateWithLifecycle()
    val activity = LocalContext.current as Activity

    LaunchedEffect(Unit) { updateHelper.checkForUpdate() }

    when (val state = updateState) {
        is AppUpdateHelper.UpdateState.Available -> {
            Card(
                Modifier.fillMaxWidth().padding(16.dp),
                colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer)
            ) {
                Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.SystemUpdate, null)
                    Spacer(Modifier.width(8.dp))
                    Column(Modifier.weight(1f)) {
                        Text("アップデートがあります", style = MaterialTheme.typography.titleSmall)
                    }
                    Button(onClick = {
                        updateHelper.startUpdate(activity,
                            if (state.isImmediate) AppUpdateType.IMMEDIATE else AppUpdateType.FLEXIBLE)
                    }) { Text("更新") }
                }
            }
        }
        is AppUpdateHelper.UpdateState.Downloading -> {
            LinearProgressIndicator(
                progress = { state.progress },
                modifier = Modifier.fillMaxWidth().padding(16.dp)
            )
        }
        AppUpdateHelper.UpdateState.Downloaded -> {
            Snackbar(
                action = { TextButton(onClick = { updateHelper.completeUpdate() }) { Text("再起動") } }
            ) { Text("アップデートをダウンロードしました") }
        }
        else -> {}
    }
}
```

---

## まとめ

| 更新タイプ | 特徴 |
|-----------|------|
| 即時更新 | 全画面ブロック、セキュリティ修正向け |
| 柔軟更新 | バックグラウンドDL、通常更新向け |

- `AppUpdateManager`で更新チェック/実行
- 即時更新: セキュリティパッチなど緊急時
- 柔軟更新: 通常の機能更新
- `completeUpdate()`でインストール完了

---

8種類のAndroidアプリテンプレート（自動更新対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [リリースチェックリスト](https://zenn.dev/myougatheaxo/articles/android-app-release-checklist-2026)
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [App Bundle](https://zenn.dev/myougatheaxo/articles/android-compose-app-bundle-delivery-2026)
