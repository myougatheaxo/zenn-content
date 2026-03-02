---
title: "Accompanist Permissions完全ガイド — 権限リクエスト/複数権限/理由表示"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "permissions"]
published: true
---

## この記事で学べること

**Accompanist Permissions**（rememberPermissionState、複数権限リクエスト、根拠説明UI）を解説します。

---

## 単一権限

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CameraPermissionScreen() {
    val cameraPermission = rememberPermissionState(Manifest.permission.CAMERA)

    when {
        cameraPermission.status.isGranted -> {
            Text("カメラ権限: 許可済み ✓", color = Color(0xFF4CAF50))
        }
        cameraPermission.status.shouldShowRationale -> {
            Column(Modifier.padding(16.dp)) {
                Text("カメラ権限が必要です", style = MaterialTheme.typography.titleMedium)
                Text("写真撮影のためにカメラへのアクセスが必要です。")
                Spacer(Modifier.height(8.dp))
                Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                    Text("権限を許可する")
                }
            }
        }
        else -> {
            Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                Icon(Icons.Default.Camera, null)
                Spacer(Modifier.width(8.dp))
                Text("カメラを使用")
            }
        }
    }
}
```

---

## 複数権限

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun MultiplePermissionsScreen() {
    val permissions = rememberMultiplePermissionsState(
        listOf(
            Manifest.permission.CAMERA,
            Manifest.permission.RECORD_AUDIO,
            Manifest.permission.ACCESS_FINE_LOCATION
        )
    )

    Column(Modifier.padding(16.dp)) {
        if (permissions.allPermissionsGranted) {
            Text("全権限が許可されています", color = Color(0xFF4CAF50))
        } else {
            Text("以下の権限が必要です:", style = MaterialTheme.typography.titleMedium)
            Spacer(Modifier.height(8.dp))

            permissions.permissions.forEach { perm ->
                val name = when (perm.permission) {
                    Manifest.permission.CAMERA -> "カメラ"
                    Manifest.permission.RECORD_AUDIO -> "マイク"
                    Manifest.permission.ACCESS_FINE_LOCATION -> "位置情報"
                    else -> perm.permission
                }
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(
                        if (perm.status.isGranted) Icons.Default.CheckCircle else Icons.Default.Cancel,
                        null,
                        tint = if (perm.status.isGranted) Color(0xFF4CAF50) else Color.Red
                    )
                    Spacer(Modifier.width(8.dp))
                    Text(name)
                }
            }

            Spacer(Modifier.height(16.dp))
            Button(onClick = { permissions.launchMultiplePermissionRequest() }) {
                Text("権限を許可する")
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

    if (!permission.status.isGranted && !permission.status.shouldShowRationale) {
        // 「今後表示しない」を選択済み → 設定画面へ誘導
        Card(Modifier.padding(16.dp)) {
            Column(Modifier.padding(16.dp)) {
                Text("カメラ権限が拒否されています", style = MaterialTheme.typography.titleMedium)
                Text("設定画面から手動で許可してください。")
                Spacer(Modifier.height(8.dp))
                OutlinedButton(onClick = {
                    context.startActivity(Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                        data = Uri.fromParts("package", context.packageName, null)
                    })
                }) {
                    Text("設定を開く")
                }
            }
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
| `shouldShowRationale` | 理由説明必要判定 |
| `launchPermissionRequest` | 権限リクエスト |

- `rememberPermissionState`でCompose内で権限管理
- `shouldShowRationale`で理由説明UIを表示
- 複数権限を一括リクエスト可能
- 永久拒否時は設定画面へ誘導

---

8種類のAndroidアプリテンプレート（権限管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [パーミッション管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [カメラプレビュー](https://zenn.dev/myougatheaxo/articles/android-compose-compose-camera-preview-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
