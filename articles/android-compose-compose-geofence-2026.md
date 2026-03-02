---
title: "Compose Geofence完全ガイド — GeofencingClient/トリガー/通知/バックグラウンド"
emoji: "📍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "location"]
published: true
---

## この記事で学べること

**Compose Geofence**（GeofencingClient、ジオフェンストリガー、バックグラウンド通知、Compose UI連携）を解説します。

---

## Geofence登録

```kotlin
class GeofenceManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val geofencingClient = LocationServices.getGeofencingClient(context)

    @SuppressLint("MissingPermission")
    fun addGeofence(id: String, lat: Double, lng: Double, radius: Float = 100f) {
        val geofence = Geofence.Builder()
            .setRequestId(id)
            .setCircularRegion(lat, lng, radius)
            .setExpirationDuration(Geofence.NEVER_EXPIRE)
            .setTransitionTypes(
                Geofence.GEOFENCE_TRANSITION_ENTER or Geofence.GEOFENCE_TRANSITION_EXIT
            )
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

        geofencingClient.addGeofences(request, pendingIntent)
    }

    fun removeGeofence(id: String) {
        geofencingClient.removeGeofences(listOf(id))
    }
}
```

---

## BroadcastReceiver

```kotlin
class GeofenceBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val event = GeofencingEvent.fromIntent(intent) ?: return
        if (event.hasError()) return

        val transition = event.geofenceTransition
        val ids = event.triggeringGeofences?.map { it.requestId } ?: return

        val message = when (transition) {
            Geofence.GEOFENCE_TRANSITION_ENTER -> "エリアに入りました: ${ids.joinToString()}"
            Geofence.GEOFENCE_TRANSITION_EXIT -> "エリアから出ました: ${ids.joinToString()}"
            else -> return
        }

        NotificationHelper(context).show("ジオフェンス", message)
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun GeofenceScreen(geofenceManager: GeofenceManager = hiltViewModel<GeofenceViewModel>().manager) {
    var geofences by remember { mutableStateOf(listOf<GeofenceData>()) }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("ジオフェンス管理", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        Button(onClick = {
            geofenceManager.addGeofence("office", 35.6812, 139.7671, 200f)
            geofences = geofences + GeofenceData("office", 35.6812, 139.7671, 200f)
        }) { Text("オフィスを追加") }

        Spacer(Modifier.height(16.dp))

        LazyColumn {
            items(geofences) { fence ->
                Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                    ListItem(
                        headlineContent = { Text(fence.id) },
                        supportingContent = { Text("半径: ${fence.radius}m") },
                        trailingContent = {
                            IconButton(onClick = {
                                geofenceManager.removeGeofence(fence.id)
                                geofences = geofences - fence
                            }) { Icon(Icons.Default.Delete, "削除") }
                        }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GeofencingClient` | ジオフェンス管理 |
| `Geofence.Builder` | フェンス定義 |
| `BroadcastReceiver` | トリガー受信 |
| `BACKGROUND_LOCATION` | バックグラウンド許可 |

- `GeofencingClient`でジオフェンスを登録・削除
- `ENTER`/`EXIT`トリガーで通知やアクションを実行
- Android 12+は`ACCESS_BACKGROUND_LOCATION`が必要
- バッテリー効率の良い位置ベーストリガー

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Location](https://zenn.dev/myougatheaxo/articles/android-compose-compose-location-2026)
- [Compose Map](https://zenn.dev/myougatheaxo/articles/android-compose-compose-map-2026)
- [Compose WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-workmanager-2026)
