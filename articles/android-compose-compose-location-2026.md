---
title: "Compose Location完全ガイド — FusedLocationProvider/位置情報取得/パーミッション"
emoji: "📍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "location"]
published: true
---

## この記事で学べること

**Compose Location**（FusedLocationProviderClient、位置情報取得、パーミッション処理、位置更新）を解説します。

---

## パーミッション+位置取得

```kotlin
@Composable
fun LocationScreen() {
    val context = LocalContext.current
    var location by remember { mutableStateOf<Location?>(null) }

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        if (permissions.values.all { it }) {
            getCurrentLocation(context) { location = it }
        }
    }

    LaunchedEffect(Unit) {
        permissionLauncher.launch(arrayOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        ))
    }

    Column(Modifier.padding(16.dp)) {
        location?.let {
            Text("緯度: ${it.latitude}")
            Text("経度: ${it.longitude}")
            Text("精度: ${it.accuracy}m")
        } ?: Text("位置情報を取得中...")
    }
}

@SuppressLint("MissingPermission")
fun getCurrentLocation(context: Context, onLocation: (Location) -> Unit) {
    val client = LocationServices.getFusedLocationProviderClient(context)
    client.getCurrentLocation(Priority.PRIORITY_HIGH_ACCURACY, null)
        .addOnSuccessListener { location -> location?.let(onLocation) }
}
```

---

## 位置更新の監視

```kotlin
@SuppressLint("MissingPermission")
@Composable
fun LocationUpdates() {
    val context = LocalContext.current
    var location by remember { mutableStateOf<Location?>(null) }

    DisposableEffect(Unit) {
        val client = LocationServices.getFusedLocationProviderClient(context)
        val request = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 5000L)
            .setMinUpdateDistanceMeters(10f)
            .build()

        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                location = result.lastLocation
            }
        }

        client.requestLocationUpdates(request, callback, Looper.getMainLooper())

        onDispose {
            client.removeLocationUpdates(callback)
        }
    }

    Column(Modifier.padding(16.dp)) {
        Text("リアルタイム位置", style = MaterialTheme.typography.titleMedium)
        location?.let {
            Text("${it.latitude}, ${it.longitude}")
        } ?: CircularProgressIndicator()
    }
}
```

---

## 距離計算

```kotlin
@Composable
fun DistanceCalculator(targetLat: Double, targetLng: Double) {
    var currentLocation by remember { mutableStateOf<Location?>(null) }

    // 位置取得（省略）

    currentLocation?.let { current ->
        val results = FloatArray(1)
        Location.distanceBetween(current.latitude, current.longitude, targetLat, targetLng, results)
        val distanceKm = results[0] / 1000

        Text("目的地まで: ${"%.1f".format(distanceKm)} km")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FusedLocationProviderClient` | 位置取得 |
| `getCurrentLocation` | 現在地1回取得 |
| `requestLocationUpdates` | 継続的位置更新 |
| `Location.distanceBetween` | 距離計算 |

- パーミッション処理が必須（FINE/COARSE_LOCATION）
- `DisposableEffect`で`removeLocationUpdates`を忘れずに
- `Priority.PRIORITY_HIGH_ACCURACY`でGPS精度
- `setMinUpdateDistanceMeters`で不要な更新を抑制

---

8種類のAndroidアプリテンプレート（位置情報対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose GoogleMap](https://zenn.dev/myougatheaxo/articles/android-compose-compose-google-map-2026)
- [Compose PermissionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [Compose Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-2026)
