---
title: "Compose Widget完全ガイド — Glance/ホーム画面ウィジェット/データ更新"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "glance"]
published: true
---

## この記事で学べること

**Compose Widget**（Glance、ホーム画面ウィジェット、GlanceAppWidget、データ更新）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("androidx.glance:glance-appwidget:1.1.0")
    implementation("androidx.glance:glance-material3:1.1.0")
}
```

---

## 基本ウィジェット

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
    val tasks = listOf("買い物", "掃除", "レポート作成")

    Column(
        modifier = GlanceModifier.fillMaxSize()
            .background(GlanceTheme.colors.surface)
            .padding(16.dp)
    ) {
        Text("今日のタスク",
            style = TextStyle(fontWeight = FontWeight.Bold,
                color = GlanceTheme.colors.onSurface, fontSize = 16.sp))
        Spacer(GlanceModifier.height(8.dp))

        tasks.forEach { task ->
            Row(GlanceModifier.fillMaxWidth().padding(vertical = 4.dp),
                verticalAlignment = Alignment.CenterVertically) {
                CheckBox(checked = false, onCheckedChange = actionRunCallback<ToggleAction>())
                Spacer(GlanceModifier.width(8.dp))
                Text(task, style = TextStyle(color = GlanceTheme.colors.onSurface))
            }
        }
    }
}

class ToggleAction : ActionCallback {
    override suspend fun onAction(context: Context, glanceId: GlanceId,
        parameters: ActionParameters) {
        // チェック状態を更新
        TodoWidget().update(context, glanceId)
    }
}
```

---

## WidgetReceiver + Manifest

```kotlin
class TodoWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget = TodoWidget()
}
```

```xml
<!-- AndroidManifest.xml -->
<receiver android:name=".TodoWidgetReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data android:name="android.appwidget.provider"
               android:resource="@xml/todo_widget_info" />
</receiver>
```

```xml
<!-- res/xml/todo_widget_info.xml -->
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="250dp"
    android:minHeight="110dp"
    android:targetCellWidth="3"
    android:targetCellHeight="2"
    android:updatePeriodMillis="3600000"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen" />
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GlanceAppWidget` | ウィジェット定義 |
| `GlanceTheme` | M3テーマ対応 |
| `GlanceModifier` | Glance用Modifier |
| `actionRunCallback` | クリックアクション |

- GlanceでCompose風にウィジェットを構築
- `GlanceTheme`でMaterial3テーマに対応
- `actionRunCallback`でユーザー操作を処理
- `updatePeriodMillis`で自動更新間隔を設定

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Shortcuts](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shortcuts-2026)
- [Compose AlarmManager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-alarm-manager-2026)
- [WorkManager Basic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-basic-2026)
