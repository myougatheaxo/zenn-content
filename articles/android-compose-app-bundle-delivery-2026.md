---
title: "Android App Bundle完全ガイド — Play Feature Delivery/動的モジュール"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "appbundle"]
published: true
---

## この記事で学べること

**Android App Bundle**（AAB形式、Play Feature Delivery、動的モジュール、条件付き配信、サイズ最適化）を解説します。

---

## App Bundleとは

```kotlin
// build.gradle.kts (app)
android {
    bundle {
        language {
            enableSplit = true  // 言語リソース分割
        }
        density {
            enableSplit = true  // 画面密度分割
        }
        abi {
            enableSplit = true  // CPUアーキテクチャ分割
        }
    }
}
```

Google PlayがデバイスごとにAPKを最適化配信。平均20%のサイズ削減。

---

## Play Feature Delivery

```kotlin
// 動的機能モジュール: build.gradle.kts
plugins {
    id("com.android.dynamic-feature")
}
android {
    namespace = "com.example.feature.camera"
}
dependencies {
    implementation(project(":app"))
}
```

```xml
<!-- feature/camera/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:dist="http://schemas.android.com/apk/distribution">
    <dist:module
        dist:instant="false"
        dist:title="@string/title_camera">
        <dist:delivery>
            <dist:on-demand />
        </dist:delivery>
    </dist:module>
</manifest>
```

---

## 動的モジュールのインストール

```kotlin
class DynamicFeatureManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val splitInstallManager = SplitInstallManagerFactory.create(context)

    fun installModule(
        moduleName: String,
        onProgress: (Int) -> Unit,
        onSuccess: () -> Unit,
        onFailure: (Exception) -> Unit
    ) {
        val request = SplitInstallRequest.newBuilder()
            .addModule(moduleName)
            .build()

        splitInstallManager.startInstall(request)
            .addOnSuccessListener { sessionId ->
                splitInstallManager.registerListener { state ->
                    when (state.status()) {
                        SplitInstallSessionStatus.INSTALLED -> onSuccess()
                        SplitInstallSessionStatus.DOWNLOADING -> {
                            val progress = (state.bytesDownloaded() * 100 / state.totalBytesToDownload()).toInt()
                            onProgress(progress)
                        }
                        SplitInstallSessionStatus.FAILED -> {
                            onFailure(Exception("Install failed: ${state.errorCode()}"))
                        }
                        else -> {}
                    }
                }
            }
            .addOnFailureListener { onFailure(it) }
    }

    fun isModuleInstalled(moduleName: String): Boolean {
        return splitInstallManager.installedModules.contains(moduleName)
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun DynamicFeatureScreen(
    featureManager: DynamicFeatureManager,
    moduleName: String,
    installedContent: @Composable () -> Unit
) {
    var installState by remember { mutableStateOf(
        if (featureManager.isModuleInstalled(moduleName)) InstallState.Installed
        else InstallState.NotInstalled
    )}
    var progress by remember { mutableIntStateOf(0) }

    when (installState) {
        InstallState.NotInstalled -> {
            Column(
                Modifier.fillMaxSize().padding(16.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text("この機能は追加ダウンロードが必要です")
                Spacer(Modifier.height(16.dp))
                Button(onClick = {
                    installState = InstallState.Installing
                    featureManager.installModule(
                        moduleName,
                        onProgress = { progress = it },
                        onSuccess = { installState = InstallState.Installed },
                        onFailure = { installState = InstallState.NotInstalled }
                    )
                }) { Text("ダウンロード") }
            }
        }
        InstallState.Installing -> {
            Column(
                Modifier.fillMaxSize(),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                CircularProgressIndicator()
                Spacer(Modifier.height(8.dp))
                Text("ダウンロード中: $progress%")
                LinearProgressIndicator(progress = { progress / 100f })
            }
        }
        InstallState.Installed -> installedContent()
    }
}

enum class InstallState { NotInstalled, Installing, Installed }
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| APK分割 | `bundle { enableSplit = true }` |
| オンデマンド配信 | `<dist:on-demand />` |
| インストール管理 | `SplitInstallManager` |
| 進捗表示 | `SplitInstallSessionStatus` |

- AAB形式でデバイスごとに最適化APK配信
- 動的モジュールで初回DLサイズ削減
- `SplitInstallManager`でモジュール管理
- 条件付き配信で国/API別配信も可能

---

8種類のAndroidアプリテンプレート（App Bundle最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [R8/ProGuard](https://zenn.dev/myougatheaxo/articles/android-compose-r8-proguard-2026)
- [リリースチェックリスト](https://zenn.dev/myougatheaxo/articles/android-app-release-checklist-2026)
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
