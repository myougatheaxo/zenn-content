---
title: "Glance APIでホーム画面ウィジェットを作る — Compose風のWidget開発"
emoji: "🧩"
type: "tech"
topics: ["android", "kotlin", "widget", "glance"]
published: true
---

## この記事で学べること

Android 12+の**Glance API**を使えば、Compose風の記法でホーム画面ウィジェットを開発できます。RemoteViewsの複雑さから解放されます。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.glance:glance-appwidget:1.1.0")
    implementation("androidx.glance:glance-material3:1.1.0")
}
```

---

## 基本のウィジェット

```kotlin
class HabitWidget : GlanceAppWidget() {
    override suspend fun provideGlance(
        context: Context,
        id: GlanceId
    ) {
        provideContent {
            GlanceTheme {
                HabitWidgetContent()
            }
        }
    }
}

@Composable
fun HabitWidgetContent() {
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(GlanceTheme.colors.surface)
            .cornerRadius(16.dp)
            .padding(16.dp)
    ) {
        Text(
            text = "今日の習慣",
            style = TextStyle(
                fontWeight = FontWeight.Bold,
                fontSize = 16.sp,
                color = GlanceTheme.colors.onSurface
            )
        )

        Spacer(GlanceModifier.height(8.dp))

        Text(
            text = "3 / 5 完了",
            style = TextStyle(
                fontSize = 24.sp,
                color = GlanceTheme.colors.primary
            )
        )
    }
}
```

---

## WidgetReceiver

```kotlin
class HabitWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = HabitWidget()
}
```

---

## AndroidManifest.xml

```xml
<receiver
    android:name=".widget.HabitWidgetReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/habit_widget_info" />
</receiver>
```

---

## ウィジェット設定XML

```xml
<!-- res/xml/habit_widget_info.xml -->
<appwidget-provider
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="180dp"
    android:minHeight="110dp"
    android:targetCellWidth="3"
    android:targetCellHeight="2"
    android:updatePeriodMillis="3600000"
    android:initialLayout="@layout/glance_default_loading_layout"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen"
    android:description="@string/widget_description"
    android:previewImage="@drawable/widget_preview" />
```

---

## ボタンアクション

```kotlin
@Composable
fun CounterWidget() {
    var count by remember { mutableStateOf(0) }

    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(GlanceTheme.colors.surface)
            .padding(16.dp)
    ) {
        Text("Count: $count")

        Button(
            text = "+1",
            onClick = actionRunCallback<IncrementAction>()
        )
    }
}

class IncrementAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // DataStoreで状態を更新
        HabitWidget().update(context, glanceId)
    }
}
```

---

## DataStoreとの連携

```kotlin
class HabitWidget : GlanceAppWidget() {
    override suspend fun provideGlance(
        context: Context,
        id: GlanceId
    ) {
        val prefs = context.habitDataStore.data.first()
        val completedCount = prefs[COMPLETED_KEY] ?: 0
        val totalCount = prefs[TOTAL_KEY] ?: 5

        provideContent {
            GlanceTheme {
                HabitWidgetContent(completedCount, totalCount)
            }
        }
    }
}
```

---

## Glance vs RemoteViews

| 項目 | Glance | RemoteViews |
|------|--------|-------------|
| 記法 | Compose風 | XML + RemoteViews API |
| 学習コスト | 低い | 高い |
| 柔軟性 | 制限あり | 制限あり（別の制約） |
| Material3対応 | glance-material3 | 手動 |
| 最小API | 26 | 21 |

---

## まとめ

- **Glance API**でCompose風にウィジェット開発
- `GlanceAppWidget` + `GlanceAppWidgetReceiver`が基本
- `GlanceTheme`でMaterial3対応
- `actionRunCallback`でボタンアクション
- DataStoreと連携してデータ表示

---

8種類のAndroidアプリテンプレート（ウィジェット追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-workmanager-guide-2026)
- [DataStore移行ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-migration-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
