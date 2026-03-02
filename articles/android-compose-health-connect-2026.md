---
title: "Health Connect API完全ガイド — 歩数/心拍/睡眠データ"
emoji: "❤️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "health"]
published: true
---

## この記事で学べること

**Health Connect API**（歩数データ、心拍数、睡眠記録、パーミッション、Compose統合）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.health.connect:connect-client:1.1.0-alpha10")
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.health.READ_STEPS" />
<uses-permission android:name="android.permission.health.READ_HEART_RATE" />
<uses-permission android:name="android.permission.health.READ_SLEEP" />
<uses-permission android:name="android.permission.health.WRITE_STEPS" />

<activity>
    <intent-filter>
        <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
    </intent-filter>
</activity>
```

---

## HealthConnectRepository

```kotlin
class HealthRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val client = HealthConnectClient.getOrCreate(context)

    // パーミッション確認
    suspend fun hasPermissions(): Boolean {
        val granted = client.permissionController.getGrantedPermissions()
        return REQUIRED_PERMISSIONS.all { it in granted }
    }

    // 歩数読み取り
    suspend fun readSteps(start: Instant, end: Instant): Long {
        val response = client.aggregate(
            AggregateRequest(
                metrics = setOf(StepsRecord.COUNT_TOTAL),
                timeRangeFilter = TimeRangeFilter.between(start, end)
            )
        )
        return response[StepsRecord.COUNT_TOTAL] ?: 0
    }

    // 心拍数読み取り
    suspend fun readHeartRate(start: Instant, end: Instant): List<HeartRateData> {
        val response = client.readRecords(
            ReadRecordsRequest(
                recordType = HeartRateRecord::class,
                timeRangeFilter = TimeRangeFilter.between(start, end)
            )
        )
        return response.records.flatMap { record ->
            record.samples.map { sample ->
                HeartRateData(sample.time, sample.beatsPerMinute)
            }
        }
    }

    // 睡眠データ読み取り
    suspend fun readSleep(start: Instant, end: Instant): List<SleepData> {
        val response = client.readRecords(
            ReadRecordsRequest(
                recordType = SleepSessionRecord::class,
                timeRangeFilter = TimeRangeFilter.between(start, end)
            )
        )
        return response.records.map { record ->
            SleepData(
                startTime = record.startTime,
                endTime = record.endTime,
                duration = Duration.between(record.startTime, record.endTime)
            )
        }
    }

    // 歩数書き込み
    suspend fun writeSteps(steps: Long, start: Instant, end: Instant) {
        client.insertRecords(listOf(
            StepsRecord(
                count = steps,
                startTime = start,
                endTime = end,
                startZoneOffset = ZoneOffset.systemDefault().rules.getOffset(start),
                endZoneOffset = ZoneOffset.systemDefault().rules.getOffset(end)
            )
        ))
    }

    companion object {
        val REQUIRED_PERMISSIONS = setOf(
            HealthPermission.getReadPermission(StepsRecord::class),
            HealthPermission.getReadPermission(HeartRateRecord::class),
            HealthPermission.getReadPermission(SleepSessionRecord::class)
        )
    }
}

data class HeartRateData(val time: Instant, val bpm: Long)
data class SleepData(val startTime: Instant, val endTime: Instant, val duration: Duration)
```

---

## Compose画面

```kotlin
@Composable
fun HealthDashboard(viewModel: HealthViewModel = hiltViewModel()) {
    val steps by viewModel.todaySteps.collectAsStateWithLifecycle()
    val heartRate by viewModel.latestHeartRate.collectAsStateWithLifecycle()
    val sleep by viewModel.lastNightSleep.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("ヘルスダッシュボード", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        HealthCard(
            icon = Icons.Default.DirectionsWalk,
            label = "今日の歩数",
            value = "%,d 歩".format(steps)
        )

        HealthCard(
            icon = Icons.Default.FavoriteBorder,
            label = "心拍数",
            value = "${heartRate ?: "--"} bpm"
        )

        HealthCard(
            icon = Icons.Default.Bedtime,
            label = "昨夜の睡眠",
            value = sleep?.let { "${it.toHours()}時間${it.toMinutesPart()}分" } ?: "--"
        )
    }
}

@Composable
fun HealthCard(icon: ImageVector, label: String, value: String) {
    Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
        Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
            Icon(icon, null, Modifier.size(40.dp), tint = MaterialTheme.colorScheme.primary)
            Spacer(Modifier.width(16.dp))
            Column {
                Text(label, style = MaterialTheme.typography.bodySmall)
                Text(value, style = MaterialTheme.typography.headlineSmall)
            }
        }
    }
}
```

---

## まとめ

| データ | Record型 | 集計 |
|--------|---------|------|
| 歩数 | `StepsRecord` | `COUNT_TOTAL` |
| 心拍 | `HeartRateRecord` | `BPM_AVG` |
| 睡眠 | `SleepSessionRecord` | `SLEEP_DURATION_TOTAL` |

- Health Connect APIで統一的なヘルスデータアクセス
- `readRecords`/`aggregate`で読み取り
- パーミッション管理が必須
- Google Fit後継の標準API

---

8種類のAndroidアプリテンプレート（ヘルス機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Wear OS Compose](https://zenn.dev/myougatheaxo/articles/android-compose-wear-os-basics-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-chain-2026)
