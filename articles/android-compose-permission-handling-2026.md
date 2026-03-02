---
title: "Composeパーミッション処理ガイド — カメラ/位置情報/通知"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "permission"]
published: true
---

## この記事で学べること

Composeでの**ランタイムパーミッション処理**（Accompanist不要、公式API）を解説します。

---

## 単一パーミッション

```kotlin
@Composable
fun CameraScreen() {
    val cameraPermissionState = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            // カメラ起動
        }
    }

    val context = LocalContext.current

    Column(
        modifier = Modifier.fillMaxSize(),
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
                    cameraPermissionState.launch(Manifest.permission.CAMERA)
                }
            }
        }) {
            Text("カメラを開く")
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
    val context = LocalContext.current

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        hasPermission = permissions.values.all { it }
    }

    LaunchedEffect(Unit) {
        val fineLocation = ContextCompat.checkSelfPermission(
            context, Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED

        val coarseLocation = ContextCompat.checkSelfPermission(
            context, Manifest.permission.ACCESS_COARSE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED

        if (fineLocation && coarseLocation) {
            hasPermission = true
        } else {
            permissionLauncher.launch(
                arrayOf(
                    Manifest.permission.ACCESS_FINE_LOCATION,
                    Manifest.permission.ACCESS_COARSE_LOCATION
                )
            )
        }
    }

    if (hasPermission) {
        Text("位置情報を取得中...")
    } else {
        Text("位置情報パーミッションが必要です")
    }
}
```

---

## 通知パーミッション (Android 13+)

```kotlin
@Composable
fun NotificationPermission() {
    val context = LocalContext.current
    var granted by remember { mutableStateOf(false) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        granted = isGranted
    }

    LaunchedEffect(Unit) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            val status = ContextCompat.checkSelfPermission(
                context, Manifest.permission.POST_NOTIFICATIONS
            )
            if (status != PackageManager.PERMISSION_GRANTED) {
                launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
            } else {
                granted = true
            }
        } else {
            granted = true // Android 12以下は不要
        }
    }

    if (granted) {
        Text("通知が有効です")
    }
}
```

---

## パーミッション拒否時の説明UI

```kotlin
@Composable
fun PermissionRationale(
    onRequestPermission: () -> Unit,
    onOpenSettings: () -> Unit
) {
    val context = LocalContext.current

    AlertDialog(
        onDismissRequest = {},
        title = { Text("パーミッションが必要です") },
        text = {
            Text("この機能を使うにはカメラへのアクセスが必要です。設定から許可してください。")
        },
        confirmButton = {
            TextButton(onClick = {
                val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                    data = Uri.fromParts("package", context.packageName, null)
                }
                context.startActivity(intent)
            }) {
                Text("設定を開く")
            }
        },
        dismissButton = {
            TextButton(onClick = onRequestPermission) {
                Text("再リクエスト")
            }
        }
    )
}
```

---

## まとめ

- `rememberLauncherForActivityResult`でパーミッション要求
- `RequestPermission`で単一、`RequestMultiplePermissions`で複数
- `ContextCompat.checkSelfPermission`で事前チェック
- Android 13+は`POST_NOTIFICATIONS`パーミッション必須
- 拒否時は設定画面への誘導UIを表示
- `shouldShowRequestPermissionRationale`で説明表示判定

---

8種類のAndroidアプリテンプレート（パーミッション処理実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カメラ/QRスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-qrcode-scanner-2026)
- [位置情報/地図連携](https://zenn.dev/myougatheaxo/articles/android-compose-map-location-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
