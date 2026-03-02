---
title: "callbackFlow完全ガイド — コールバック→Flow変換/センサー/位置情報"
emoji: "📞"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**callbackFlow**（コールバックAPIのFlow変換、センサー/位置情報/ネットワーク監視）を解説します。

---

## callbackFlow基本

```kotlin
fun Context.connectivityFlow(): Flow<Boolean> = callbackFlow {
    val cm = getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    val callback = object : ConnectivityManager.NetworkCallback() {
        override fun onAvailable(network: Network) { trySend(true) }
        override fun onLost(network: Network) { trySend(false) }
    }
    cm.registerDefaultNetworkCallback(callback)
    // 初期値
    val isConnected = cm.activeNetwork != null
    trySend(isConnected)

    awaitClose { cm.unregisterNetworkCallback(callback) }
}

@Composable
fun ConnectivityStatus() {
    val context = LocalContext.current
    val isConnected by context.connectivityFlow().collectAsStateWithLifecycle(initialValue = true)

    if (!isConnected) {
        Card(colors = CardDefaults.cardColors(containerColor = Color(0xFFFFF3E0))) {
            Row(Modifier.padding(12.dp)) {
                Icon(Icons.Default.WifiOff, null, tint = Color(0xFFFF9800))
                Spacer(Modifier.width(8.dp))
                Text("オフラインです")
            }
        }
    }
}
```

---

## 位置情報

```kotlin
@SuppressLint("MissingPermission")
fun FusedLocationProviderClient.locationFlow(
    interval: Long = 10000
): Flow<Location> = callbackFlow {
    val request = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, interval).build()
    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.lastLocation?.let { trySend(it) }
        }
    }
    requestLocationUpdates(request, callback, Looper.getMainLooper())
    awaitClose { removeLocationUpdates(callback) }
}
```

---

## SharedPreferences監視

```kotlin
fun SharedPreferences.asFlow(key: String): Flow<String?> = callbackFlow {
    val listener = SharedPreferences.OnSharedPreferenceChangeListener { prefs, changedKey ->
        if (changedKey == key) trySend(prefs.getString(key, null))
    }
    registerOnSharedPreferenceChangeListener(listener)
    trySend(getString(key, null)) // 初期値

    awaitClose { unregisterOnSharedPreferenceChangeListener(listener) }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `callbackFlow` | コールバック→Flow |
| `trySend` | 値の送信 |
| `awaitClose` | クリーンアップ |
| `collectAsStateWithLifecycle` | Compose連携 |

- `callbackFlow`でコールバックAPIをFlowに変換
- `trySend`で非suspend関数から値送信
- `awaitClose`でリスナー解除を保証
- ネットワーク/位置情報/センサー監視に活用

---

8種類のAndroidアプリテンプレート（リアクティブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [SharedFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-shared-flow-2026)
- [StateFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-state-flow-2026)
- [接続状態](https://zenn.dev/myougatheaxo/articles/android-compose-compose-connectivity-2026)
