---
title: "Compose HealthConnect完全ガイド — 歩数/心拍数/データ読み書き/権限管理"
emoji: "❤️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "healthconnect"]
published: true
---

## この記事で学べること

**Compose HealthConnect**（Health Connect API、歩数・心拍データ、パーミッション、データ読み書き）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("androidx.health.connect:connect-client:1.1.0-alpha07")
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.health.READ_STEPS" />
<uses-permission android:name="android.permission.health.READ_HEART_RATE" />
<uses-permission android:name="android.permission.health.WRITE_STEPS" />

<application>
    <activity>
        <intent-filter>
            <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
        </intent-filter>
    </activity>
</application>
```

---

## 歩数データ読み取り

```kotlin
@Composable
fun StepsScreen() {
    val context = LocalContext.current
    var steps by remember { mutableLongStateOf(0L) }
    var hasPermission by remember { mutableStateOf(false) }

    val permissionLauncher = rememberLauncherForActivityResult(
        PermissionController.createRequestPermissionResultContract()
    ) { granted ->
        hasPermission = granted.containsAll(
            setOf(HealthPermission.getReadPermission(StepsRecord::class))
        )
    }

    LaunchedEffect(Unit) {
        val client = HealthConnectClient.getOrCreate(context)
        val permissions = setOf(HealthPermission.getReadPermission(StepsRecord::class))
        val granted = client.permissionController.getGrantedPermissions()
        if (granted.containsAll(permissions)) {
            hasPermission = true
            steps = readSteps(client)
        } else {
            permissionLauncher.launch(permissions)
        }
    }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        Text("今日の歩数", style = MaterialTheme.typography.titleMedium)
        Text("$steps", style = MaterialTheme.typography.displayLarge)
        Text("歩", style = MaterialTheme.typography.titleLarge)
    }
}

suspend fun readSteps(client: HealthConnectClient): Long {
    val now = Instant.now()
    val startOfDay = now.atZone(ZoneId.systemDefault())
        .toLocalDate().atStartOfDay(ZoneId.systemDefault()).toInstant()
    val response = client.readRecords(
        ReadRecordsRequest(StepsRecord::class, TimeRangeFilter.between(startOfDay, now))
    )
    return response.records.sumOf { it.count }
}
```

---

## データ書き込み

```kotlin
suspend fun writeSteps(client: HealthConnectClient, count: Long) {
    val now = Instant.now()
    val record = StepsRecord(
        count = count,
        startTime = now.minus(Duration.ofHours(1)),
        endTime = now,
        startZoneOffset = ZoneOffset.systemDefault().rules.getOffset(now),
        endZoneOffset = ZoneOffset.systemDefault().rules.getOffset(now)
    )
    client.insertRecords(listOf(record))
}

@Composable
fun WriteStepsScreen() {
    val context = LocalContext.current
    var stepInput by remember { mutableStateOf("") }
    val scope = rememberCoroutineScope()

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        OutlinedTextField(
            value = stepInput, onValueChange = { stepInput = it },
            label = { Text("歩数を入力") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number)
        )
        Button(onClick = {
            scope.launch {
                val client = HealthConnectClient.getOrCreate(context)
                writeSteps(client, stepInput.toLongOrNull() ?: 0)
            }
        }) { Text("記録する") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `HealthConnectClient` | Health Connect接続 |
| `StepsRecord` | 歩数データ |
| `HeartRateRecord` | 心拍データ |
| `PermissionController` | 権限管理 |

- Health ConnectはGoogleのヘルスデータ統合API
- `readRecords`/`insertRecords`でデータ読み書き
- `TimeRangeFilter`で期間指定
- パーミッション管理はManifest+ランタイム両方必要

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Permissions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permissions-2026)
- [Compose BatteryState](https://zenn.dev/myougatheaxo/articles/android-compose-compose-battery-state-2026)
- [Compose ForegroundService](https://zenn.dev/myougatheaxo/articles/android-compose-compose-foreground-service-2026)
