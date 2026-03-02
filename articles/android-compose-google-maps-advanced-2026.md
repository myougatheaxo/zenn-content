---
title: "Google Maps + Compose応用ガイド — クラスタリング/カスタムマーカー/ジオコーディング"
emoji: "🗺️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googlemaps"]
published: true
---

## この記事で学べること

**Google Maps + Compose応用**（マーカークラスタリング、カスタムマーカー、ジオコーディング、ルート描画、ヒートマップ）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.google.maps.android:maps-compose:6.2.1")
    implementation("com.google.maps.android:maps-compose-utils:6.2.1")
    implementation("com.google.android.gms:play-services-maps:19.0.0")
}
```

---

## カスタムマーカー

```kotlin
@Composable
fun CustomMarkerMap(spots: List<Spot>) {
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(
            LatLng(35.6812, 139.7671), 12f // 東京
        )
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        spots.forEach { spot ->
            MarkerComposable(
                keys = arrayOf(spot.id),
                state = MarkerState(position = LatLng(spot.lat, spot.lng))
            ) {
                // ComposeでカスタムマーカーUI
                Card(
                    colors = CardDefaults.cardColors(
                        containerColor = MaterialTheme.colorScheme.primaryContainer
                    ),
                    shape = RoundedCornerShape(8.dp)
                ) {
                    Text(
                        spot.name,
                        Modifier.padding(horizontal = 8.dp, vertical = 4.dp),
                        style = MaterialTheme.typography.labelSmall
                    )
                }
            }
        }
    }
}
```

---

## マーカークラスタリング

```kotlin
@Composable
fun ClusteredMap(items: List<SpotClusterItem>) {
    val cameraPositionState = rememberCameraPositionState()

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        Clustering(
            items = items,
            onClusterClick = { cluster ->
                // クラスタクリック: ズームイン
                cameraPositionState.move(
                    CameraUpdateFactory.newLatLngZoom(cluster.position, 15f)
                )
                true
            },
            onClusterItemClick = { item ->
                // 個別マーカークリック
                false
            },
            clusterContent = { cluster ->
                Box(
                    Modifier.size(40.dp).background(
                        MaterialTheme.colorScheme.primary, CircleShape
                    ),
                    contentAlignment = Alignment.Center
                ) {
                    Text("${cluster.size}", color = Color.White, fontWeight = FontWeight.Bold)
                }
            }
        )
    }
}

data class SpotClusterItem(
    val spot: Spot,
    val itemPosition: LatLng = LatLng(spot.lat, spot.lng),
    val itemTitle: String = spot.name
) : ClusterItem {
    override fun getPosition() = itemPosition
    override fun getTitle() = itemTitle
    override fun getSnippet() = spot.description
    override fun getZIndex() = 0f
}
```

---

## ポリライン（ルート描画）

```kotlin
@Composable
fun RouteMap(routePoints: List<LatLng>) {
    val cameraPositionState = rememberCameraPositionState()

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        Polyline(
            points = routePoints,
            color = MaterialTheme.colorScheme.primary,
            width = 8f,
            pattern = listOf(Dot(), Gap(10f))
        )

        if (routePoints.isNotEmpty()) {
            Marker(state = MarkerState(routePoints.first()), title = "出発")
            Marker(state = MarkerState(routePoints.last()), title = "到着")
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| カスタムマーカー | `MarkerComposable` |
| クラスタリング | `Clustering` |
| ルート描画 | `Polyline` |
| 円/多角形 | `Circle` / `Polygon` |
| カメラ操作 | `CameraPositionState` |

- `MarkerComposable`でComposeウィジェットをマーカーに
- `Clustering`で大量マーカーをグループ化
- `Polyline`でルートや経路を描画
- `maps-compose`でGoogle MapsをCompose完全統合

---

8種類のAndroidアプリテンプレート（Maps連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Maps基本](https://zenn.dev/myougatheaxo/articles/android-compose-google-maps-2026)
- [位置情報/ジオフェンス](https://zenn.dev/myougatheaxo/articles/android-compose-location-geofence-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
