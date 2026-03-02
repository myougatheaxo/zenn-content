---
title: "Google Maps Compose実装ガイド — マーカー/現在地/ジオコーディング"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googlemap"]
published: true
---

## この記事で学べること

**Google Maps Compose**のマーカー、現在地表示、ジオコーディングを解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.maps.android:maps-compose:6.2.1")
    implementation("com.google.android.gms:play-services-maps:19.0.0")
    implementation("com.google.android.gms:play-services-location:21.3.0")
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<application>
    <meta-data
        android:name="com.google.android.geo.API_KEY"
        android:value="${MAPS_API_KEY}" />
</application>
```

---

## 基本の地図表示

```kotlin
@Composable
fun BasicMap() {
    val tokyo = LatLng(35.6812, 139.7671)
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(tokyo, 12f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        properties = MapProperties(isMyLocationEnabled = false),
        uiSettings = MapUiSettings(zoomControlsEnabled = true)
    ) {
        Marker(
            state = MarkerState(position = tokyo),
            title = "東京駅",
            snippet = "Tokyo Station"
        )
    }
}
```

---

## 複数マーカー

```kotlin
data class MapPlace(
    val name: String,
    val position: LatLng,
    val description: String
)

@Composable
fun MultiMarkerMap(places: List<MapPlace>) {
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(
            places.firstOrNull()?.position ?: LatLng(35.6812, 139.7671),
            11f
        )
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        places.forEach { place ->
            Marker(
                state = MarkerState(position = place.position),
                title = place.name,
                snippet = place.description
            )
        }
    }
}
```

---

## 現在地の取得

```kotlin
@Composable
fun CurrentLocationMap() {
    val context = LocalContext.current
    var currentLocation by remember { mutableStateOf<LatLng?>(null) }
    val cameraPositionState = rememberCameraPositionState()

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        if (permissions.values.all { it }) {
            getCurrentLocation(context) { location ->
                currentLocation = LatLng(location.latitude, location.longitude)
            }
        }
    }

    LaunchedEffect(Unit) {
        permissionLauncher.launch(
            arrayOf(
                Manifest.permission.ACCESS_FINE_LOCATION,
                Manifest.permission.ACCESS_COARSE_LOCATION
            )
        )
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
            Marker(
                state = MarkerState(position = it),
                title = "現在地"
            )
        }
    }
}

private fun getCurrentLocation(context: Context, onLocation: (Location) -> Unit) {
    val client = LocationServices.getFusedLocationProviderClient(context)
    client.lastLocation.addOnSuccessListener { location ->
        location?.let { onLocation(it) }
    }
}
```

---

## カメラアニメーション

```kotlin
@Composable
fun AnimatedCameraMap(places: List<MapPlace>) {
    val cameraPositionState = rememberCameraPositionState()
    var selectedIndex by remember { mutableIntStateOf(0) }
    val scope = rememberCoroutineScope()

    Column {
        GoogleMap(
            modifier = Modifier
                .fillMaxWidth()
                .weight(1f),
            cameraPositionState = cameraPositionState
        ) {
            places.forEach { place ->
                Marker(
                    state = MarkerState(position = place.position),
                    title = place.name
                )
            }
        }

        LazyRow(
            contentPadding = PaddingValues(8.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            itemsIndexed(places) { index, place ->
                FilterChip(
                    selected = selectedIndex == index,
                    onClick = {
                        selectedIndex = index
                        scope.launch {
                            cameraPositionState.animate(
                                CameraUpdateFactory.newLatLngZoom(place.position, 15f),
                                durationMs = 1000
                            )
                        }
                    },
                    label = { Text(place.name) }
                )
            }
        }
    }
}
```

---

## まとめ

- `GoogleMap`コンポーザブルで地図表示
- `Marker`/`MarkerState`でマーカー配置
- `rememberCameraPositionState`でカメラ位置管理
- `FusedLocationProviderClient`で現在地取得
- `cameraPositionState.animate()`でカメラアニメーション
- パーミッション処理は`rememberLauncherForActivityResult`

---

8種類のAndroidアプリテンプレート（位置情報対応設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Maps Composeガイド](https://zenn.dev/myougatheaxo/articles/android-map-compose-2026)
- [パーミッション管理ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handler-2026)
- [Foreground Serviceガイド](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
