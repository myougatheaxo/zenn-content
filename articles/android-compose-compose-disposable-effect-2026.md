---
title: "DisposableEffect完全ガイド — リソース管理/リスナー登録/クリーンアップ"
emoji: "🧹"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lifecycle"]
published: true
---

## この記事で学べること

**DisposableEffect**（リソース管理、リスナー登録/解除、onDispose、ライフサイクル監視）を解説します。

---

## DisposableEffect基本

```kotlin
@Composable
fun LifecycleObserver(onResume: () -> Unit, onPause: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> onResume()
                Lifecycle.Event.ON_PAUSE -> onPause()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}
```

---

## BroadcastReceiver

```kotlin
@Composable
fun BatteryMonitor() {
    val context = LocalContext.current
    var batteryLevel by remember { mutableIntStateOf(0) }

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                batteryLevel = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, 0)
            }
        }
        context.registerReceiver(receiver, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
        onDispose { context.unregisterReceiver(receiver) }
    }

    Text("バッテリー: $batteryLevel%")
}
```

---

## MapView管理

```kotlin
@Composable
fun MapViewComposable() {
    val context = LocalContext.current
    val mapView = remember { MapView(context) }
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_CREATE -> mapView.onCreate(Bundle())
                Lifecycle.Event.ON_START -> mapView.onStart()
                Lifecycle.Event.ON_RESUME -> mapView.onResume()
                Lifecycle.Event.ON_PAUSE -> mapView.onPause()
                Lifecycle.Event.ON_STOP -> mapView.onStop()
                Lifecycle.Event.ON_DESTROY -> mapView.onDestroy()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
            mapView.onDestroy()
        }
    }

    AndroidView(factory = { mapView }, Modifier.fillMaxSize())
}
```

---

## まとめ

| パターン | 登録 | 解除 |
|---------|------|------|
| LifecycleObserver | `addObserver` | `removeObserver` |
| BroadcastReceiver | `registerReceiver` | `unregisterReceiver` |
| Listener | `addListener` | `removeListener` |
| AndroidView | `onCreate` | `onDestroy` |

- `DisposableEffect`でリソースの登録/解除をペアで管理
- `onDispose`で必ずクリーンアップ
- keyが変わると旧onDispose→新Effect実行
- メモリリーク防止に必須

---

8種類のAndroidアプリテンプレート（ライフサイクル対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LaunchedEffect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-launched-effect-2026)
- [Side Effect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effect-2026)
- [Lifecycle Compose](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-compose-2026)
