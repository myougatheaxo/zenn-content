---
title: "パーミッション処理ガイド — Composeでランタイムパーミッションを扱う"
emoji: "🛡️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "permission"]
published: true
---

## この記事で学べること

Composeでの**ランタイムパーミッション**のリクエストと状態管理を解説します。

---

## 依存関係

```kotlin
dependencies {
    implementation("com.google.accompanist:accompanist-permissions:0.36.0")
}
```

---

## 単一パーミッション

```kotlin
@Composable
fun CameraScreen() {
    val context = LocalContext.current

    val cameraPermissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            // カメラ起動
        } else {
            // 拒否時の処理
        }
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Button(onClick = {
            when {
                ContextCompat.checkSelfPermission(
                    context, Manifest.permission.CAMERA
                ) == PackageManager.PERMISSION_GRANTED -> {
                    // 既に許可済み
                }
                else -> {
                    cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
                }
            }
        }) {
            Text("カメラを起動")
        }
    }
}
```

---

## 複数パーミッション

```kotlin
@Composable
fun LocationScreen() {
    var hasPermission by remember { mutableStateOf(false) }

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        hasPermission = permissions.values.all { it }
    }

    if (hasPermission) {
        MapContent()
    } else {
        Column(
            Modifier.fillMaxSize().padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text("位置情報の許可が必要です")
            Spacer(Modifier.height(16.dp))
            Button(onClick = {
                permissionLauncher.launch(
                    arrayOf(
                        Manifest.permission.ACCESS_FINE_LOCATION,
                        Manifest.permission.ACCESS_COARSE_LOCATION
                    )
                )
            }) {
                Text("許可する")
            }
        }
    }
}
```

---

## Accompanist Permissions

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun AccompanistExample() {
    val cameraPermission = rememberPermissionState(Manifest.permission.CAMERA)

    when {
        cameraPermission.status.isGranted -> {
            CameraPreview()
        }
        cameraPermission.status.shouldShowRationale -> {
            // 理由の説明
            Column(
                Modifier.fillMaxSize().padding(16.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text("カメラはQRコード読み取りに必要です")
                Spacer(Modifier.height(16.dp))
                Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                    Text("許可する")
                }
            }
        }
        else -> {
            // 初回リクエスト or 永続拒否
            Column(
                Modifier.fillMaxSize().padding(16.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text("カメラの許可が必要です")
                Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                    Text("許可する")
                }
            }
        }
    }
}
```

---

## 設定画面への誘導

```kotlin
@Composable
fun PermissionDeniedView(permission: String) {
    val context = LocalContext.current

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            Icons.Default.Block,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.error
        )
        Spacer(Modifier.height(16.dp))
        Text(
            "パーミッションが拒否されています",
            style = MaterialTheme.typography.headlineSmall
        )
        Spacer(Modifier.height(8.dp))
        Text(
            "設定画面から手動で許可してください",
            style = MaterialTheme.typography.bodyMedium
        )
        Spacer(Modifier.height(24.dp))
        Button(onClick = {
            val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                data = Uri.fromParts("package", context.packageName, null)
            }
            context.startActivity(intent)
        }) {
            Text("設定を開く")
        }
    }
}
```

---

## まとめ

- `rememberLauncherForActivityResult`で標準パーミッションリクエスト
- `RequestPermission`で単一、`RequestMultiplePermissions`で複数
- `shouldShowRationale`で理由説明の表示判定
- Accompanist Permissionsで宣言的なパーミッション管理
- 永続拒否時は設定画面への誘導が必要
- Android 13+の`POST_NOTIFICATIONS`等の新パーミッションに注意

---

8種類のAndroidアプリテンプレート（パーミッション処理設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CameraX実装ガイド](https://zenn.dev/myougatheaxo/articles/android-camerax-compose-2026)
- [生体認証ガイド](https://zenn.dev/myougatheaxo/articles/android-biometric-auth-2026)
- [Google Maps Composeガイド](https://zenn.dev/myougatheaxo/articles/android-map-compose-2026)
