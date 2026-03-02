---
title: "Google Maps + Compose — 地図表示・マーカー・現在地の実装ガイド"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googlemap"]
published: true
---

## この記事で学べること

**Maps Compose**ライブラリを使って、Composeアプリに地図表示・マーカー・現在地機能を実装する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.maps.android:maps-compose:6.1.0")
    implementation("com.google.android.gms:play-services-maps:19.0.0")
    implementation("com.google.android.gms:play-services-location:21.3.0")
}
```

```xml
<!-- AndroidManifest.xml -->
<meta-data
    android:name="com.google.android.geo.API_KEY"
    android:value="${MAPS_API_KEY}" />

<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

```properties
# local.properties
MAPS_API_KEY=your_api_key_here
```

---

## 基本の地図表示

```kotlin
@Composable
fun BasicMap() {
    val tokyo = LatLng(35.6812, 139.7671)
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(tokyo, 15f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    )
}
```

---

## マーカー

```kotlin
@Composable
fun MapWithMarkers() {
    val spots = listOf(
        LatLng(35.6812, 139.7671) to "東京駅",
        LatLng(35.6586, 139.7454) to "東京タワー",
        LatLng(35.7101, 139.8107) to "東京スカイツリー"
    )

    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(spots[0].first, 12f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        spots.forEach { (position, title) ->
            Marker(
                state = MarkerState(position = position),
                title = title,
                snippet = "タップで詳細"
            )
        }
    }
}
```

---

## カスタムマーカー

```kotlin
@Composable
fun CustomMarkerMap() {
    GoogleMap(modifier = Modifier.fillMaxSize()) {
        MarkerComposable(
            state = MarkerState(position = LatLng(35.6812, 139.7671))
        ) {
            // Composableをマーカーとして使用
            Card(
                colors = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                ),
                shape = RoundedCornerShape(8.dp)
            ) {
                Text(
                    "¥1,500",
                    modifier = Modifier.padding(horizontal = 8.dp, vertical = 4.dp),
                    style = MaterialTheme.typography.labelMedium
                )
            }
        }
    }
}
```

---

## 現在地表示

```kotlin
@Composable
fun CurrentLocationMap() {
    val context = LocalContext.current
    val fusedLocationClient = remember {
        LocationServices.getFusedLocationProviderClient(context)
    }
    val cameraPositionState = rememberCameraPositionState()

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            fusedLocationClient.lastLocation.addOnSuccessListener { location ->
                location?.let {
                    cameraPositionState.position = CameraPosition.fromLatLngZoom(
                        LatLng(it.latitude, it.longitude), 15f
                    )
                }
            }
        }
    }

    LaunchedEffect(Unit) {
        permissionLauncher.launch(Manifest.permission.ACCESS_FINE_LOCATION)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        properties = MapProperties(isMyLocationEnabled = true),
        uiSettings = MapUiSettings(myLocationButtonEnabled = true)
    )
}
```

---

## ポリライン・円

```kotlin
GoogleMap(modifier = Modifier.fillMaxSize()) {
    // ポリライン（ルート表示）
    Polyline(
        points = routePoints,
        color = Color.Blue,
        width = 8f
    )

    // 円（範囲表示）
    Circle(
        center = LatLng(35.6812, 139.7671),
        radius = 500.0,  // メートル
        fillColor = Color.Blue.copy(alpha = 0.1f),
        strokeColor = Color.Blue,
        strokeWidth = 2f
    )
}
```

---

## まとめ

- `GoogleMap` composableで地図表示
- `Marker` / `MarkerComposable`でマーカー配置
- `rememberCameraPositionState`でカメラ制御
- `FusedLocationProviderClient`で現在地取得
- `MapProperties`で地図設定（現在地、地図タイプ）
- `Polyline` / `Circle`で図形描画

---

8種類のAndroidアプリテンプレート（地図機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [パーミッション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-permission-runtime-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
