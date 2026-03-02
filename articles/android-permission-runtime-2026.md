---
title: "Android 14のパーミッション完全ガイド — カメラ・位置情報・通知の正しいリクエスト"
emoji: "🔐"
type: "tech"
topics: ["android", "kotlin", "permission", "security"]
published: true
---

## この記事で学べること

Android 6.0以降、危険なパーミッションは**実行時にリクエスト**が必要です。Android 13で通知パーミッション、Android 14で写真/動画のアクセスが変更されました。Composeでの正しい実装方法を解説します。

---

## 基本パターン

```kotlin
@Composable
fun CameraPermissionScreen() {
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            // カメラを使う処理
        } else {
            // 拒否された場合の処理
        }
    }

    Button(onClick = {
        launcher.launch(Manifest.permission.CAMERA)
    }) {
        Text("カメラを起動")
    }
}
```

---

## パーミッションの状態確認

```kotlin
@Composable
fun PermissionAwareButton() {
    val context = LocalContext.current
    val hasPermission = ContextCompat.checkSelfPermission(
        context, Manifest.permission.CAMERA
    ) == PackageManager.PERMISSION_GRANTED

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) openCamera()
    }

    Button(onClick = {
        if (hasPermission) {
            openCamera()
        } else {
            launcher.launch(Manifest.permission.CAMERA)
        }
    }) {
        Text(if (hasPermission) "カメラを開く" else "カメラの許可をリクエスト")
    }
}
```

---

## 複数パーミッション

```kotlin
val launcher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    val cameraGranted = permissions[Manifest.permission.CAMERA] ?: false
    val locationGranted = permissions[Manifest.permission.ACCESS_FINE_LOCATION] ?: false

    if (cameraGranted && locationGranted) {
        // 両方許可された
    }
}

Button(onClick = {
    launcher.launch(arrayOf(
        Manifest.permission.CAMERA,
        Manifest.permission.ACCESS_FINE_LOCATION
    ))
}) {
    Text("カメラと位置情報を許可")
}
```

---

## 理由の説明（Rationale）

```kotlin
@Composable
fun LocationPermissionWithRationale() {
    val context = LocalContext.current
    val activity = context as Activity
    var showRationale by remember { mutableStateOf(false) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (!granted) {
            // 永久拒否の可能性 → 設定画面へ誘導
        }
    }

    Button(onClick = {
        when {
            ContextCompat.checkSelfPermission(
                context, Manifest.permission.ACCESS_FINE_LOCATION
            ) == PackageManager.PERMISSION_GRANTED -> {
                // 既に許可済み
            }
            activity.shouldShowRequestPermissionRationale(
                Manifest.permission.ACCESS_FINE_LOCATION
            ) -> {
                showRationale = true
            }
            else -> {
                launcher.launch(Manifest.permission.ACCESS_FINE_LOCATION)
            }
        }
    }) {
        Text("現在地を取得")
    }

    if (showRationale) {
        AlertDialog(
            onDismissRequest = { showRationale = false },
            title = { Text("位置情報が必要です") },
            text = { Text("近くのお店を表示するために位置情報を使用します。") },
            confirmButton = {
                TextButton(onClick = {
                    showRationale = false
                    launcher.launch(Manifest.permission.ACCESS_FINE_LOCATION)
                }) { Text("許可する") }
            },
            dismissButton = {
                TextButton(onClick = { showRationale = false }) { Text("後で") }
            }
        )
    }
}
```

---

## 通知パーミッション（Android 13+）

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

```kotlin
@Composable
fun NotificationPermission() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        val launcher = rememberLauncherForActivityResult(
            ActivityResultContracts.RequestPermission()
        ) { /* ... */ }

        LaunchedEffect(Unit) {
            launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
}
```

---

## 永久拒否 → 設定画面へ誘導

```kotlin
fun openAppSettings(context: Context) {
    val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
        data = Uri.fromParts("package", context.packageName, null)
    }
    context.startActivity(intent)
}
```

「二度と表示しない」を選ばれた場合、アプリの設定画面からしか許可できません。

---

## AndroidManifest.xml

```xml
<!-- カメラ -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="false" />

<!-- 位置情報 -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- 通知 (Android 13+) -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

`uses-feature`に`required="false"`を付けると、カメラなしの端末でもインストール可能。

---

## まとめ

- 危険なパーミッションは`rememberLauncherForActivityResult`でリクエスト
- `shouldShowRequestPermissionRationale`で理由説明ダイアログ
- 永久拒否時は設定画面へ誘導
- Android 13+の通知パーミッションを忘れずに

---

8種類のAndroidアプリテンプレート（パーミッション不要設計・トラッキングなし）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [Android通知の実装ガイド](https://zenn.dev/myougatheaxo/articles/android-notifications-guide-2026)
- [アクセシビリティ対応入門](https://zenn.dev/myougatheaxo/articles/android-accessibility-basics-2026)
