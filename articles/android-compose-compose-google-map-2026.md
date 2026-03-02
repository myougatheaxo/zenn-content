---
title: "Compose GoogleMap完全ガイド — Maps Compose/マーカー/ポリライン"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googlemap"]
published: true
---

## この記事で学べること

**Compose GoogleMap**（Maps Compose SDK、マーカー、ポリライン、カメラ制御）を解説します。

---

## 基本GoogleMap

```kotlin
// build.gradle
// implementation "com.google.maps.android:maps-compose:4.3.0"

@Composable
fun MapScreen() {
    val tokyo = LatLng(35.6812, 139.7671)
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(tokyo, 12f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
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

## 複数マーカー+情報ウィンドウ

```kotlin
@Composable
fun MultiMarkerMap(locations: List<MapLocation>) {
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(
            LatLng(locations.first().lat, locations.first().lng), 10f
        )
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        uiSettings = MapUiSettings(zoomControlsEnabled = true)
    ) {
        locations.forEach { loc ->
            MarkerInfoWindowContent(
                state = MarkerState(position = LatLng(loc.lat, loc.lng))
            ) {
                Column(Modifier.padding(8.dp)) {
                    Text(loc.name, fontWeight = FontWeight.Bold)
                    Text(loc.address, style = MaterialTheme.typography.bodySmall)
                }
            }
        }
    }
}

data class MapLocation(val name: String, val address: String, val lat: Double, val lng: Double)
```

---

## ポリライン+円

```kotlin
@Composable
fun RouteMap(route: List<LatLng>, center: LatLng) {
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(center, 14f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        Polyline(
            points = route,
            color = Color.Blue,
            width = 8f
        )

        Circle(
            center = center,
            radius = 500.0,
            fillColor = Color.Blue.copy(alpha = 0.1f),
            strokeColor = Color.Blue,
            strokeWidth = 2f
        )

        Marker(state = MarkerState(position = route.first()), title = "出発")
        Marker(state = MarkerState(position = route.last()), title = "到着")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GoogleMap` | 地図表示 |
| `Marker` | マーカー |
| `Polyline` | 経路線 |
| `Circle` | 円エリア |

- Maps Compose SDKでComposeネイティブな地図表示
- `rememberCameraPositionState`でカメラ位置管理
- `MarkerInfoWindowContent`でカスタム情報ウィンドウ
- API Keyの設定が必要（AndroidManifest.xml）

---

8種類のAndroidアプリテンプレート（地図対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Location](https://zenn.dev/myougatheaxo/articles/android-compose-compose-location-2026)
- [Compose PermissionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
