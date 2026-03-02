---
title: "App Widget (Glance)完全ガイド — ホーム画面ウィジェット"
emoji: "🔲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "widget"]
published: true
---

## この記事で学べること

**Jetpack Glance**（App Widget、GlanceAppWidget、GlanceTheme、データ更新）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.glance:glance-appwidget:1.1.1")
    implementation("androidx.glance:glance-material3:1.1.1")
}
```

---

## 基本ウィジェット

```kotlin
class TaskWidget : GlanceAppWidget() {

    override suspend fun provideGlance(context: Context, id: GlanceId) {
        val tasks = TaskRepository(context).getTopTasks(5)

        provideContent {
            GlanceTheme {
                TaskWidgetContent(tasks)
            }
        }
    }
}

@Composable
fun TaskWidgetContent(tasks: List<Task>) {
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .padding(16.dp)
            .background(GlanceTheme.colors.surface)
            .cornerRadius(16.dp)
    ) {
        Text(
            text = "タスク一覧",
            style = TextStyle(
                fontWeight = FontWeight.Bold,
                fontSize = 16.sp,
                color = GlanceTheme.colors.onSurface
            )
        )

        Spacer(GlanceModifier.height(8.dp))

        tasks.forEach { task ->
            Row(
                modifier = GlanceModifier
                    .fillMaxWidth()
                    .padding(vertical = 4.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                CheckBox(
                    checked = task.isCompleted,
                    onCheckedChange = actionRunCallback<ToggleTaskAction>(
                        parameters = actionParametersOf(taskIdKey to task.id)
                    )
                )
                Spacer(GlanceModifier.width(8.dp))
                Text(
                    text = task.title,
                    style = TextStyle(
                        color = GlanceTheme.colors.onSurface,
                        textDecoration = if (task.isCompleted)
                            TextDecoration.LineThrough else TextDecoration.None
                    )
                )
            }
        }

        Spacer(GlanceModifier.defaultWeight())

        Button(
            text = "アプリを開く",
            onClick = actionStartActivity<MainActivity>(),
            modifier = GlanceModifier.fillMaxWidth()
        )
    }
}

val taskIdKey = ActionParameters.Key<Int>("task_id")
```

---

## ActionCallback

```kotlin
class ToggleTaskAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        val taskId = parameters[taskIdKey] ?: return

        // DB更新
        TaskRepository(context).toggleTask(taskId)

        // ウィジェット更新
        TaskWidget().update(context, glanceId)
    }
}

class RefreshAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        TaskWidget().update(context, glanceId)
    }
}
```

---

## GlanceAppWidgetReceiver

```kotlin
class TaskWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = TaskWidget()
}
```

```xml
<!-- AndroidManifest.xml -->
<receiver
    android:name=".TaskWidgetReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/task_widget_info" />
</receiver>
```

```xml
<!-- res/xml/task_widget_info.xml -->
<appwidget-provider
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="250dp"
    android:minHeight="180dp"
    android:targetCellWidth="3"
    android:targetCellHeight="3"
    android:updatePeriodMillis="3600000"
    android:initialLayout="@layout/glance_default_loading_layout"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen"
    android:description="@string/widget_description"
    android:previewImage="@drawable/widget_preview" />
```

---

## データ更新パターン

```kotlin
// アプリ側でデータ変更時にウィジェット更新
class TaskRepository(private val context: Context) {

    suspend fun completeTask(taskId: Int) {
        dao.toggleCompleted(taskId)

        // 全ウィジェット更新
        TaskWidget().updateAll(context)
    }
}

// WorkManagerで定期更新
class WidgetUpdateWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        TaskWidget().updateAll(applicationContext)
        return Result.success()
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `GlanceAppWidget` | ウィジェット定義 |
| `GlanceAppWidgetReceiver` | ブロードキャストレシーバー |
| `GlanceTheme` | Material3テーマ |
| `ActionCallback` | ユーザー操作処理 |
| `actionParametersOf` | パラメータ受け渡し |

- GlanceはCompose風APIでウィジェット構築
- `ActionCallback`でウィジェット内操作処理
- `updateAll()`でデータ変更時に全ウィジェット更新
- Material3テーマでシステムと統一感

---

8種類のAndroidアプリテンプレート（ウィジェット対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-chain-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
