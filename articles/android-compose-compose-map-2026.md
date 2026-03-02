---
title: "Compose Map完全ガイド — GoogleMap/マーカー/ポリライン/カメラ制御"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googlemap"]
published: true
---

## この記事で学べること

**Compose Map**（GoogleMap Compose、マーカー、ポリライン、カメラ制御、クラスタリング）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation("com.google.maps.android:maps-compose:4.3.3")
    implementation("com.google.android.gms:play-services-maps:18.2.0")
}
```

```xml
<!-- AndroidManifest.xml -->
<meta-data android:name="com.google.android.geo.API_KEY"
    android:value="${MAPS_API_KEY}" />
```

---

## 基本的なMap表示

```kotlin
@Composable
fun BasicMapScreen() {
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

## 複数マーカーとポリライン

```kotlin
@Composable
fun RouteMapScreen() {
    val points = listOf(
        LatLng(35.6812, 139.7671) to "東京",
        LatLng(35.6586, 139.7454) to "渋谷",
        LatLng(35.6938, 139.7034) to "新宿"
    )
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(points.first().first, 13f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        points.forEach { (pos, title) ->
            Marker(state = MarkerState(position = pos), title = title)
        }
        Polyline(
            points = points.map { it.first },
            color = Color.Blue,
            width = 5f
        )
    }
}
```

---

## カメラ制御

```kotlin
@Composable
fun CameraControlMap() {
    val cameraPositionState = rememberCameraPositionState()
    val scope = rememberCoroutineScope()
    val locations = mapOf(
        "東京" to LatLng(35.6812, 139.7671),
        "大阪" to LatLng(34.6937, 135.5023),
        "福岡" to LatLng(33.5904, 130.4017)
    )

    Column(Modifier.fillMaxSize()) {
        LazyRow(Modifier.padding(8.dp), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            items(locations.entries.toList()) { (name, pos) ->
                AssistChip(
                    onClick = {
                        scope.launch {
                            cameraPositionState.animate(
                                CameraUpdateFactory.newLatLngZoom(pos, 14f), 1000
                            )
                        }
                    },
                    label = { Text(name) }
                )
            }
        }
        GoogleMap(
            modifier = Modifier.weight(1f),
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
| `Marker` | マーカー配置 |
| `Polyline` | 線描画 |
| `CameraPositionState` | カメラ制御 |

- `maps-compose`ライブラリでComposeネイティブのMap表示
- `Marker`/`Polyline`/`Circle`等をComposable内で宣言
- `cameraPositionState.animate()`でカメラアニメーション
- APIキーは`local.properties`で管理

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Location](https://zenn.dev/myougatheaxo/articles/android-compose-compose-location-2026)
- [Compose Geofence](https://zenn.dev/myougatheaxo/articles/android-compose-compose-geofence-2026)
- [Compose Permission](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-2026)
