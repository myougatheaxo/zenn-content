---
title: "Google Maps + Compose完全ガイド — マーカー/ポリライン/クラスタ"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googlemaps"]
published: true
---

## この記事で学べること

**Google Maps Compose**（マーカー、ポリライン、ジオフェンス、クラスタリング）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.maps.android:maps-compose:4.4.1")
    implementation("com.google.android.gms:play-services-maps:18.2.0")
    implementation("com.google.maps.android:maps-compose-utils:4.4.1")
}

// AndroidManifest.xml
// <meta-data
//     android:name="com.google.android.geo.API_KEY"
//     android:value="${MAPS_API_KEY}" />
```

---

## 基本の地図表示

```kotlin
@Composable
fun BasicMapScreen() {
    val tokyo = LatLng(35.6812, 139.7671)
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(tokyo, 12f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        properties = MapProperties(
            isMyLocationEnabled = true,
            mapType = MapType.NORMAL
        ),
        uiSettings = MapUiSettings(
            zoomControlsEnabled = true,
            myLocationButtonEnabled = true
        )
    ) {
        Marker(
            state = MarkerState(position = tokyo),
            title = "東京駅",
            snippet = "日本の中心駅"
        )
    }
}
```

---

## 複数マーカー

```kotlin
@Composable
fun MultiMarkerMap(locations: List<MapLocation>) {
    val cameraPositionState = rememberCameraPositionState()

    // 全マーカーが見えるようにカメラ調整
    LaunchedEffect(locations) {
        if (locations.isNotEmpty()) {
            val bounds = LatLngBounds.Builder().apply {
                locations.forEach { include(LatLng(it.lat, it.lng)) }
            }.build()
            cameraPositionState.animate(
                CameraUpdateFactory.newLatLngBounds(bounds, 100)
            )
        }
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        locations.forEach { location ->
            MarkerComposable(
                state = MarkerState(position = LatLng(location.lat, location.lng)),
                title = location.name
            ) {
                // カスタムマーカーUI
                Box(
                    Modifier
                        .size(40.dp)
                        .background(MaterialTheme.colorScheme.primary, CircleShape),
                    contentAlignment = Alignment.Center
                ) {
                    Icon(Icons.Default.Place, null, tint = Color.White, modifier = Modifier.size(24.dp))
                }
            }
        }
    }
}
```

---

## ポリライン・ポリゴン

```kotlin
@Composable
fun RouteMap(routePoints: List<LatLng>) {
    val cameraPositionState = rememberCameraPositionState()

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        // ルート線
        Polyline(
            points = routePoints,
            color = Color.Blue,
            width = 8f
        )

        // 始点・終点マーカー
        if (routePoints.isNotEmpty()) {
            Marker(
                state = MarkerState(position = routePoints.first()),
                title = "出発地"
            )
            Marker(
                state = MarkerState(position = routePoints.last()),
                title = "目的地"
            )
        }

        // エリア表示
        Polygon(
            points = listOf(
                LatLng(35.68, 139.76),
                LatLng(35.69, 139.76),
                LatLng(35.69, 139.77),
                LatLng(35.68, 139.77)
            ),
            fillColor = Color.Blue.copy(alpha = 0.2f),
            strokeColor = Color.Blue,
            strokeWidth = 2f
        )
    }
}
```

---

## 現在地取得

```kotlin
@Composable
fun CurrentLocationMap() {
    val context = LocalContext.current
    var currentLocation by remember { mutableStateOf<LatLng?>(null) }
    val cameraPositionState = rememberCameraPositionState()

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            getCurrentLocation(context) { location ->
                currentLocation = LatLng(location.latitude, location.longitude)
            }
        }
    }

    LaunchedEffect(Unit) {
        permissionLauncher.launch(Manifest.permission.ACCESS_FINE_LOCATION)
    }

    LaunchedEffect(currentLocation) {
        currentLocation?.let {
            cameraPositionState.animate(
                CameraUpdateFactory.newLatLngZoom(it, 15f)
            )
        }
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        properties = MapProperties(isMyLocationEnabled = true)
    ) {
        currentLocation?.let {
            Circle(
                center = it,
                radius = 200.0,
                fillColor = Color.Blue.copy(alpha = 0.1f),
                strokeColor = Color.Blue,
                strokeWidth = 2f
            )
        }
    }
}

fun getCurrentLocation(context: Context, onLocation: (Location) -> Unit) {
    val client = LocationServices.getFusedLocationProviderClient(context)
    client.lastLocation.addOnSuccessListener { location ->
        location?.let { onLocation(it) }
    }
}
```

---

## まとめ

- `GoogleMap` ComposableでComposeネイティブな地図表示
- `rememberCameraPositionState`でカメラ制御
- `Marker`/`MarkerComposable`でマーカー（カスタムUI対応）
- `Polyline`/`Polygon`/`Circle`で図形描画
- `LatLngBounds.Builder`で全マーカー表示範囲調整
- `FusedLocationProviderClient`で現在地取得

---

8種類のAndroidアプリテンプレート（地図機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Map+Location](https://zenn.dev/myougatheaxo/articles/android-compose-map-location-2026)
- [パーミッション処理](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
- [Composeライフサイクル](https://zenn.dev/myougatheaxo/articles/android-compose-lifecycle-aware-2026)
