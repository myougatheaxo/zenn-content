---
title: "位置情報 + ジオフェンス完全ガイド — FusedLocation + Compose"
emoji: "📍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "location"]
published: true
---

## この記事で学べること

**位置情報**（FusedLocationProvider、位置追跡、ジオフェンス、パーミッション管理）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.google.android.gms:play-services-location:21.3.0")
    implementation("com.google.accompanist:accompanist-permissions:0.36.0")
}
```

---

## LocationRepository

```kotlin
class LocationRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val client = LocationServices.getFusedLocationProviderClient(context)

    // 最新位置取得
    @SuppressLint("MissingPermission")
    suspend fun getCurrentLocation(): Location? = suspendCancellableCoroutine { cont ->
        val task = client.getCurrentLocation(
            Priority.PRIORITY_HIGH_ACCURACY,
            CancellationTokenSource().token
        )
        task.addOnSuccessListener { cont.resume(it) }
        task.addOnFailureListener { cont.resumeWithException(it) }
    }

    // 位置追跡（Flow）
    @SuppressLint("MissingPermission")
    fun locationUpdates(intervalMs: Long = 5000): Flow<Location> = callbackFlow {
        val request = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, intervalMs)
            .setMinUpdateDistanceMeters(10f)
            .build()

        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { trySend(it) }
            }
        }

        client.requestLocationUpdates(request, callback, Looper.getMainLooper())
        awaitClose { client.removeLocationUpdates(callback) }
    }
}
```

---

## パーミッション管理

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun LocationPermissionScreen(content: @Composable () -> Unit) {
    val permissions = rememberMultiplePermissionsState(
        listOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        )
    )

    when {
        permissions.allPermissionsGranted -> content()
        permissions.shouldShowRationale -> {
            Column(
                Modifier.fillMaxSize().padding(24.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text("位置情報の許可が必要です")
                Text("近くのスポットを表示するために使用します")
                Spacer(Modifier.height(16.dp))
                Button(onClick = { permissions.launchMultiplePermissionRequest() }) {
                    Text("許可する")
                }
            }
        }
        else -> {
            LaunchedEffect(Unit) {
                permissions.launchMultiplePermissionRequest()
            }
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
    private val client = LocationServices.getGeofencingClient(context)

    @SuppressLint("MissingPermission")
    fun addGeofence(
        id: String,
        latitude: Double,
        longitude: Double,
        radiusMeters: Float = 100f
    ) {
        val geofence = Geofence.Builder()
            .setRequestId(id)
            .setCircularRegion(latitude, longitude, radiusMeters)
            .setTransitionTypes(
                Geofence.GEOFENCE_TRANSITION_ENTER or Geofence.GEOFENCE_TRANSITION_EXIT
            )
            .setExpirationDuration(Geofence.NEVER_EXPIRE)
            .build()

        val request = GeofencingRequest.Builder()
            .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
            .addGeofence(geofence)
            .build()

        val pendingIntent = PendingIntent.getBroadcast(
            context, 0,
            Intent(context, GeofenceBroadcastReceiver::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
        )

        client.addGeofences(request, pendingIntent)
    }
}

class GeofenceBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val event = GeofencingEvent.fromIntent(intent) ?: return
        when (event.geofenceTransition) {
            Geofence.GEOFENCE_TRANSITION_ENTER -> {
                showNotification(context, "エリアに入りました")
            }
            Geofence.GEOFENCE_TRANSITION_EXIT -> {
                showNotification(context, "エリアから出ました")
            }
        }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun LocationScreen(viewModel: LocationViewModel = hiltViewModel()) {
    val location by viewModel.currentLocation.collectAsStateWithLifecycle()

    LocationPermissionScreen {
        Column(Modifier.fillMaxSize().padding(16.dp)) {
            Text("現在地", style = MaterialTheme.typography.headlineMedium)
            Spacer(Modifier.height(16.dp))

            location?.let { loc ->
                Card(Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(16.dp)) {
                        Text("緯度: ${loc.latitude}")
                        Text("経度: ${loc.longitude}")
                        Text("精度: ${loc.accuracy}m")
                    }
                }
            } ?: CircularProgressIndicator()
        }
    }
}
```

---

## まとめ

| 機能 | クラス |
|------|--------|
| 現在地取得 | `getCurrentLocation()` |
| 位置追跡 | `requestLocationUpdates()` → `callbackFlow` |
| ジオフェンス | `GeofencingClient` |
| パーミッション | `accompanist-permissions` |

- `callbackFlow`で位置更新をFlow化
- `Priority.PRIORITY_HIGH_ACCURACY`で高精度
- ジオフェンスでエリア入退出検知
- バックグラウンド位置情報は別パーミッション

---

8種類のAndroidアプリテンプレート（位置情報対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Maps](https://zenn.dev/myougatheaxo/articles/android-compose-google-maps-2026)
- [バックグラウンドタスク](https://zenn.dev/myougatheaxo/articles/android-compose-background-task-2026)
- [ローカル通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-schedule-2026)
