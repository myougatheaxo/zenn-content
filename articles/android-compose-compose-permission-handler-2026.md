---
title: "パーミッション管理完全ガイド — rememberPermissionState/複数権限/Rationale"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "permission"]
published: true
---

## この記事で学べること

**パーミッション管理**（rememberPermissionState、複数権限リクエスト、Rationale表示）を解説します。

---

## 単一パーミッション

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CameraPermissionScreen() {
    val cameraPermission = rememberPermissionState(Manifest.permission.CAMERA)

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        when {
            cameraPermission.status.isGranted -> {
                Text("カメラ権限あり")
                CameraPreview()
            }
            cameraPermission.status.shouldShowRationale -> {
                Text("カメラは写真撮影に必要です")
                Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                    Text("権限を許可")
                }
            }
            else -> {
                Text("カメラ権限が必要です")
                Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                    Text("権限をリクエスト")
                }
            }
        }
    }
}
```

---

## 複数パーミッション

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun MultiPermissionScreen() {
    val permissions = rememberMultiplePermissionsState(
        listOf(
            Manifest.permission.CAMERA,
            Manifest.permission.RECORD_AUDIO,
            Manifest.permission.ACCESS_FINE_LOCATION
        )
    )

    Column(Modifier.padding(16.dp)) {
        if (permissions.allPermissionsGranted) {
            Text("全権限許可済み")
        } else {
            val denied = permissions.revokedPermissions
            Text("未許可: ${denied.map { it.permission.substringAfterLast('.') }}")

            Spacer(Modifier.height(8.dp))

            if (permissions.shouldShowRationale) {
                AlertDialog(
                    onDismissRequest = {},
                    title = { Text("権限が必要です") },
                    text = { Text("この機能にはカメラ、マイク、位置情報が必要です") },
                    confirmButton = {
                        TextButton(onClick = { permissions.launchMultiplePermissionRequest() }) {
                            Text("許可する")
                        }
                    }
                )
            } else {
                Button(onClick = { permissions.launchMultiplePermissionRequest() }) {
                    Text("権限をリクエスト")
                }
            }
        }
    }
}
```

---

## 設定画面誘導

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun PermissionWithSettings() {
    val context = LocalContext.current
    val permission = rememberPermissionState(Manifest.permission.CAMERA)
    var permanentlyDenied by remember { mutableStateOf(false) }

    LaunchedEffect(permission.status) {
        if (!permission.status.isGranted && !permission.status.shouldShowRationale) {
            permanentlyDenied = true
        }
    }

    if (permanentlyDenied && !permission.status.isGranted) {
        Column(Modifier.padding(16.dp)) {
            Text("カメラ権限が永久に拒否されました。設定から許可してください。")
            Button(onClick = {
                val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                    data = Uri.fromParts("package", context.packageName, null)
                }
                context.startActivity(intent)
            }) { Text("設定を開く") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `rememberPermissionState` | 単一権限管理 |
| `rememberMultiplePermissionsState` | 複数権限管理 |
| `shouldShowRationale` | 説明表示判定 |
| `launchPermissionRequest` | 権限リクエスト |

- `rememberPermissionState`で宣言的にパーミッション管理
- `shouldShowRationale`で理由説明ダイアログ表示
- 永久拒否時は設定画面への誘導
- `accompanist-permissions`ライブラリを使用

---

8種類のAndroidアプリテンプレート（パーミッション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camerax-analysis-2026)
- [位置情報](https://zenn.dev/myougatheaxo/articles/android-compose-location-map-2026)
- [Accompanist](https://zenn.dev/myougatheaxo/articles/android-compose-accompanist-2026)
