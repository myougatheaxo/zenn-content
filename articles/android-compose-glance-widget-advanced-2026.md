---
title: "Glanceウィジェット完全ガイド — カスタムUI/データ更新/Worker連携"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "widget"]
published: true
---

## この記事で学べること

**Glanceウィジェット**（カスタムレイアウト、データ更新、WorkManager連携、サイズ適応、テーマ対応）を解説します。

---

## 基本ウィジェット

```kotlin
class StatsWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        val stats = StatsRepository(context).getStats()

        provideContent {
            GlanceTheme {
                StatsWidgetContent(stats)
            }
        }
    }
}

@Composable
fun StatsWidgetContent(stats: Stats) {
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(GlanceTheme.colors.surface)
            .padding(16.dp)
            .cornerRadius(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(
            text = "今日の歩数",
            style = TextStyle(
                color = GlanceTheme.colors.onSurface,
                fontSize = 14.sp
            )
        )
        Spacer(GlanceModifier.height(4.dp))
        Text(
            text = "${stats.steps}",
            style = TextStyle(
                color = GlanceTheme.colors.primary,
                fontSize = 32.sp,
                fontWeight = FontWeight.Bold
            )
        )
        Spacer(GlanceModifier.height(8.dp))
        LinearProgressIndicator(
            progress = (stats.steps / 10000f).coerceIn(0f, 1f),
            modifier = GlanceModifier.fillMaxWidth(),
            color = ColorProvider(Color(0xFF4CAF50))
        )
    }
}
```

---

## サイズ適応

```kotlin
class AdaptiveWidget : GlanceAppWidget() {
    override val sizeMode = SizeMode.Responsive(
        setOf(
            DpSize(100.dp, 50.dp),   // Small
            DpSize(200.dp, 100.dp),  // Medium
            DpSize(300.dp, 200.dp)   // Large
        )
    )

    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            val size = LocalSize.current

            when {
                size.width < 200.dp -> CompactLayout()
                size.width < 300.dp -> MediumLayout()
                else -> ExpandedLayout()
            }
        }
    }
}
```

---

## データ更新（WorkManager連携）

```kotlin
class WidgetUpdateWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val stats = StatsRepository(applicationContext).fetchLatestStats()

        // ウィジェット更新
        StatsWidget().updateAll(applicationContext)

        return Result.success()
    }

    companion object {
        fun schedule(context: Context) {
            val request = PeriodicWorkRequestBuilder<WidgetUpdateWorker>(
                15, TimeUnit.MINUTES
            ).setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            ).build()

            WorkManager.getInstance(context)
                .enqueueUniquePeriodicWork(
                    "widget_update",
                    ExistingPeriodicWorkPolicy.KEEP,
                    request
                )
        }
    }
}
```

---

## アクション処理

```kotlin
@Composable
fun InteractiveWidget() {
    Column(
        GlanceModifier.fillMaxSize().background(GlanceTheme.colors.surface).padding(12.dp)
    ) {
        Text("タスク管理", style = TextStyle(fontSize = 16.sp, fontWeight = FontWeight.Bold))

        // ボタンアクション
        Button(
            text = "アプリを開く",
            onClick = actionStartActivity<MainActivity>()
        )

        // カスタムアクション
        Button(
            text = "更新",
            onClick = actionRunCallback<RefreshAction>()
        )
    }
}

class RefreshAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // データ更新
        StatsWidget().update(context, glanceId)
    }
}
```

---

## Receiver登録

```kotlin
class StatsWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget = StatsWidget()

    override fun onEnabled(context: Context) {
        super.onEnabled(context)
        WidgetUpdateWorker.schedule(context)
    }
}
```

```xml
<!-- AndroidManifest.xml -->
<receiver android:name=".widget.StatsWidgetReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/stats_widget_info" />
</receiver>
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| レイアウト | `GlanceAppWidget` + `@Composable` |
| サイズ適応 | `SizeMode.Responsive` |
| 更新 | `WorkManager` + `updateAll()` |
| アクション | `actionRunCallback` |

- Glance APIでCompose風にウィジェット作成
- `SizeMode.Responsive`でサイズ別レイアウト
- WorkManagerで定期データ更新
- `GlanceTheme`でダイナミックカラー対応

---

8種類のAndroidアプリテンプレート（ウィジェット対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
