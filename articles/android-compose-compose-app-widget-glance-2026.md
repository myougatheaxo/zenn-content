---
title: "Compose App Widget Glance完全ガイド — GlanceAppWidget/レイアウト/状態管理/更新"
emoji: "📲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "glance"]
published: true
---

## この記事で学べること

**Compose App Widget Glance**（GlanceAppWidget、Compose風レイアウト、状態管理、定期更新）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation("androidx.glance:glance-appwidget:1.1.0")
    implementation("androidx.glance:glance-material3:1.1.0")
}
```

---

## GlanceAppWidget

```kotlin
class TodoWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            GlanceTheme {
                TodoWidgetContent()
            }
        }
    }
}

@Composable
fun TodoWidgetContent() {
    val context = LocalContext.current
    val prefs = currentState<Preferences>()
    val todoCount = prefs[intPreferencesKey("todo_count")] ?: 0

    Column(
        modifier = GlanceModifier.fillMaxSize()
            .background(GlanceTheme.colors.surface)
            .padding(16.dp)
    ) {
        Text(
            "TODO",
            style = TextStyle(
                fontSize = 18.sp,
                fontWeight = FontWeight.Bold,
                color = GlanceTheme.colors.onSurface
            )
        )
        Spacer(GlanceModifier.height(8.dp))
        Text("残り: ${todoCount}件",
            style = TextStyle(color = GlanceTheme.colors.onSurfaceVariant))
        Spacer(GlanceModifier.height(16.dp))
        Button(
            text = "アプリを開く",
            onClick = actionStartActivity<MainActivity>(),
            modifier = GlanceModifier.fillMaxWidth()
        )
    }
}
```

---

## WidgetReceiver

```kotlin
class TodoWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget = TodoWidget()
}
```

```xml
<!-- AndroidManifest.xml -->
<receiver android:name=".TodoWidgetReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/todo_widget_info" />
</receiver>
```

```xml
<!-- res/xml/todo_widget_info.xml -->
<appwidget-provider
    android:minWidth="180dp"
    android:minHeight="110dp"
    android:updatePeriodMillis="3600000"
    android:initialLayout="@layout/glance_default_loading_layout"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen" />
```

---

## 状態更新

```kotlin
// アプリ内からWidgetを更新
suspend fun updateWidget(context: Context, todoCount: Int) {
    val glanceId = GlanceAppWidgetManager(context)
        .getGlanceIds(TodoWidget::class.java).firstOrNull() ?: return

    updateAppWidgetState(context, glanceId) { prefs ->
        prefs[intPreferencesKey("todo_count")] = todoCount
    }
    TodoWidget().update(context, glanceId)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GlanceAppWidget` | ウィジェット定義 |
| `GlanceTheme` | Material3テーマ |
| `actionStartActivity` | アクション |
| `updateAppWidgetState` | 状態更新 |

- GlanceでCompose風の構文でAppWidgetを構築
- `GlanceTheme`でMaterial3テーマ対応
- `updateAppWidgetState`でウィジェットの状態を更新
- `GlanceAppWidgetReceiver`でブロードキャスト受信

---

8種類のAndroidアプリテンプレート（Widget対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Widget](https://zenn.dev/myougatheaxo/articles/android-compose-compose-widget-2026)
- [Compose WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-workmanager-2026)
- [Compose Shortcut](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shortcut-2026)
