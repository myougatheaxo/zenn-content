---
title: "位置情報/ジオフェンス完全ガイド — FusedLocation/Geofence/権限管理"
emoji: "📍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "location"]
published: true
---

## この記事で学べること

**位置情報とジオフェンス**（FusedLocationProvider、Geofencing API、権限管理、バックグラウンド位置情報）を解説します。

---

## FusedLocationProvider

```kotlin
class LocationRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val fusedClient = LocationServices.getFusedLocationProviderClient(context)

    @SuppressLint("MissingPermission")
    fun getLocationUpdates(intervalMs: Long = 10_000L): Flow<Location> = callbackFlow {
        val request = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, intervalMs)
            .setMinUpdateDistanceMeters(10f)
            .build()

        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { trySend(it) }
            }
        }

        fusedClient.requestLocationUpdates(request, callback, Looper.getMainLooper())
        awaitClose { fusedClient.removeLocationUpdates(callback) }
    }

    @SuppressLint("MissingPermission")
    suspend fun getLastLocation(): Location? {
        return fusedClient.lastLocation.await()
    }
}
```

---

## 権限管理（Compose）

```kotlin
@Composable
fun LocationPermissionScreen(onGranted: @Composable () -> Unit) {
    val permissions = rememberMultiplePermissionsState(
        listOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        )
    )

    when {
        permissions.allPermissionsGranted -> onGranted()
        permissions.shouldShowRationale -> {
            Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
                Text("位置情報の権限が必要です", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(8.dp))
                Text("近くのスポットを表示するために位置情報を使用します")
                Spacer(Modifier.height(16.dp))
                Button(onClick = { permissions.launchMultiplePermissionRequest() }) {
                    Text("権限を許可")
                }
            }
        }
        else -> {
            LaunchedEffect(Unit) { permissions.launchMultiplePermissionRequest() }
        }
    }
}
```

---

## ジオフェンス

```kotlin
class GeofenceManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val geofencingClient = LocationServices.getGeofencingClient(context)

    @SuppressLint("MissingPermission")
    suspend fun addGeofence(id: String, lat: Double, lng: Double, radiusMeters: Float = 100f) {
        val geofence = Geofence.Builder()
            .setRequestId(id)
            .setCircularRegion(lat, lng, radiusMeters)
            .setExpirationDuration(Geofence.NEVER_EXPIRE)
            .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_ENTER or Geofence.GEOFENCE_TRANSITION_EXIT)
            .build()

        val request = GeofencingRequest.Builder()
            .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
            .addGeofence(geofence)
            .build()

        val intent = PendingIntent.getBroadcast(
            context, 0,
            Intent(context, GeofenceBroadcastReceiver::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
        )

        geofencingClient.addGeofences(request, intent).await()
    }
}

class GeofenceBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val event = GeofencingEvent.fromIntent(intent) ?: return
        if (event.hasError()) return

        when (event.geofenceTransition) {
            Geofence.GEOFENCE_TRANSITION_ENTER -> { /* エリア進入 */ }
            Geofence.GEOFENCE_TRANSITION_EXIT -> { /* エリア退出 */ }
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun LocationScreen(viewModel: LocationViewModel = hiltViewModel()) {
    val location by viewModel.currentLocation.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        location?.let { loc ->
            Text("緯度: ${loc.latitude}", style = MaterialTheme.typography.titleMedium)
            Text("経度: ${loc.longitude}")
            Text("精度: ${loc.accuracy}m")
        } ?: CircularProgressIndicator()
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 位置取得 | `FusedLocationProviderClient` |
| 継続取得 | `requestLocationUpdates` + `callbackFlow` |
| ジオフェンス | `GeofencingClient` |
| 権限 | `rememberMultiplePermissionsState` |

- `callbackFlow`で位置情報をリアクティブに取得
- ジオフェンスでエリア進入/退出を検知
- 権限管理はCompose Permissionsライブラリで簡潔に
- バックグラウンド位置情報はForeground Service必須

---

8種類のAndroidアプリテンプレート（位置情報対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Maps](https://zenn.dev/myougatheaxo/articles/android-compose-google-maps-advanced-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
- [Foreground Service](https://zenn.dev/myougatheaxo/articles/android-compose-foreground-service-2026)
