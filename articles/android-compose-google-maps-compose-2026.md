---
title: "Google Maps Compose完全ガイド — GoogleMap/Marker/カメラ/ジオフェンス"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googlemaps"]
published: true
---

## この記事で学べること

**Google Maps Compose**（GoogleMap、Marker、カメラ制御、クラスタリング）を解説します。

---

## 基本GoogleMap

```kotlin
@Composable
fun MapExample() {
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
            snippet = "JR東京駅"
        )
    }
}
```

---

## 複数マーカー

```kotlin
@Composable
fun MultiMarkerMap(locations: List<LocationItem>) {
    val cameraPositionState = rememberCameraPositionState()
    val scope = rememberCoroutineScope()

    LaunchedEffect(locations) {
        if (locations.isNotEmpty()) {
            val bounds = LatLngBounds.builder().apply {
                locations.forEach { include(LatLng(it.lat, it.lng)) }
            }.build()
            cameraPositionState.animate(CameraUpdateFactory.newLatLngBounds(bounds, 100))
        }
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        locations.forEach { location ->
            MarkerInfoWindow(
                state = MarkerState(position = LatLng(location.lat, location.lng))
            ) {
                Card(Modifier.padding(8.dp)) {
                    Column(Modifier.padding(12.dp)) {
                        Text(location.name, fontWeight = FontWeight.Bold)
                        Text(location.address, fontSize = 12.sp)
                    }
                }
            }
        }
    }
}
```

---

## カメラ移動

```kotlin
@Composable
fun CameraControlMap() {
    val cameraPositionState = rememberCameraPositionState()
    val scope = rememberCoroutineScope()

    val cities = mapOf(
        "東京" to LatLng(35.6812, 139.7671),
        "大阪" to LatLng(34.6937, 135.5023),
        "名古屋" to LatLng(35.1815, 136.9066)
    )

    Column {
        Row(Modifier.padding(8.dp), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            cities.forEach { (name, latLng) ->
                AssistChip(
                    onClick = {
                        scope.launch {
                            cameraPositionState.animate(
                                CameraUpdateFactory.newLatLngZoom(latLng, 14f),
                                durationMs = 1000
                            )
                        }
                    },
                    label = { Text(name) }
                )
            }
        }
        GoogleMap(
            modifier = Modifier.fillMaxSize(),
            cameraPositionState = cameraPositionState
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GoogleMap` | 地図表示 |
| `Marker` | ピン表示 |
| `CameraPositionState` | カメラ制御 |
| `MarkerInfoWindow` | カスタム情報窓 |

- `GoogleMap`でCompose内に地図を表示
- `Marker`/`MarkerInfoWindow`でピン+情報窓
- `CameraPositionState`でカメラのアニメーション移動
- `LatLngBounds`で複数マーカーの一括表示

---

8種類のAndroidアプリテンプレート（地図対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [位置情報](https://zenn.dev/myougatheaxo/articles/android-compose-location-map-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [Foreground Service](https://zenn.dev/myougatheaxo/articles/android-compose-foreground-service-2026)
