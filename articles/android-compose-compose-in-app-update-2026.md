---
title: "In-App Update完全ガイド — 即時更新/柔軟な更新/更新状態管理"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "playstore"]
published: true
---

## この記事で学べること

**In-App Update**（即時更新、柔軟な更新、AppUpdateManager、Compose統合）を解説します。

---

## 即時更新

```kotlin
@Composable
fun ImmediateUpdateCheck() {
    val context = LocalContext.current
    val activity = context as Activity
    val updateManager = remember { AppUpdateManagerFactory.create(context) }

    LaunchedEffect(Unit) {
        val info = updateManager.appUpdateInfo.await()
        if (info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE
            && info.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE)
        ) {
            updateManager.startUpdateFlowForResult(
                info, activity, AppUpdateOptions.newBuilder(AppUpdateType.IMMEDIATE).build(), 100
            )
        }
    }
}
```

---

## 柔軟な更新

```kotlin
@Composable
fun FlexibleUpdateScreen() {
    val context = LocalContext.current
    val activity = context as Activity
    val updateManager = remember { AppUpdateManagerFactory.create(context) }
    var downloadProgress by remember { mutableFloatStateOf(0f) }
    var showUpdateBar by remember { mutableStateOf(false) }

    val listener = remember {
        InstallStateUpdatedListener { state ->
            when (state.installStatus()) {
                InstallStatus.DOWNLOADING -> {
                    val total = state.totalBytesToDownload()
                    if (total > 0) downloadProgress = state.bytesDownloaded().toFloat() / total
                }
                InstallStatus.DOWNLOADED -> showUpdateBar = true
                else -> {}
            }
        }
    }

    LaunchedEffect(Unit) {
        updateManager.registerListener(listener)
        val info = updateManager.appUpdateInfo.await()
        if (info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE
            && info.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE)
        ) {
            updateManager.startUpdateFlowForResult(
                info, activity, AppUpdateOptions.newBuilder(AppUpdateType.FLEXIBLE).build(), 101
            )
        }
    }

    DisposableEffect(Unit) { onDispose { updateManager.unregisterListener(listener) } }

    Column(Modifier.fillMaxSize()) {
        if (downloadProgress > 0f && !showUpdateBar) {
            LinearProgressIndicator(progress = { downloadProgress }, modifier = Modifier.fillMaxWidth())
        }

        if (showUpdateBar) {
            Snackbar(
                action = {
                    TextButton(onClick = { updateManager.completeUpdate() }) {
                        Text("再起動", color = Color.White)
                    }
                },
                modifier = Modifier.padding(16.dp)
            ) { Text("アップデートの準備ができました") }
        }
    }
}
```

---

## まとめ

| 更新タイプ | 動作 |
|-----------|------|
| `IMMEDIATE` | 全画面更新（ブロッキング） |
| `FLEXIBLE` | バックグラウンドDL |
| `completeUpdate()` | アプリ再起動 |
| `InstallStateUpdatedListener` | 進捗監視 |

- 即時更新は重要なセキュリティ修正に使用
- 柔軟な更新はUXを損なわずにバックグラウンドDL
- `completeUpdate()`でダウンロード後にインストール
- 更新の優先度はPlay Consoleで設定

---

8種類のAndroidアプリテンプレート（Play Store対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [In-App Review](https://zenn.dev/myougatheaxo/articles/android-compose-compose-in-app-review-2026)
- [SplashScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-splash-screen-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
