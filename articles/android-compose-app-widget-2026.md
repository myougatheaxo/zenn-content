---
title: "Glance App Widgetガイド — ホーム画面ウィジェット実装"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "widget"]
published: true
---

## この記事で学べること

**Jetpack Glance**を使ったApp Widgetの実装を解説します。

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

## 基本のウィジェット

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
    val todos = listOf("買い物", "掃除", "レポート提出")

    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(GlanceTheme.colors.surface)
            .padding(12.dp)
            .cornerRadius(16.dp)
    ) {
        Text(
            "今日のTodo",
            style = TextStyle(
                fontWeight = FontWeight.Bold,
                fontSize = 16.sp,
                color = GlanceTheme.colors.onSurface
            )
        )
        Spacer(GlanceModifier.height(8.dp))

        todos.forEach { todo ->
            Row(
                modifier = GlanceModifier
                    .fillMaxWidth()
                    .padding(vertical = 4.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                CheckBox(
                    checked = false,
                    onCheckedChange = actionRunCallback<ToggleTodoAction>()
                )
                Spacer(GlanceModifier.width(8.dp))
                Text(todo, style = TextStyle(color = GlanceTheme.colors.onSurface))
            }
        }
    }
}
```

---

## WidgetReceiver

```kotlin
class TodoWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = TodoWidget()
}
```

```xml
<!-- AndroidManifest.xml -->
<receiver
    android:name=".widget.TodoWidgetReceiver"
    android:exported="true">
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
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="250dp"
    android:minHeight="110dp"
    android:resizeMode="horizontal|vertical"
    android:targetCellWidth="3"
    android:targetCellHeight="2"
    android:widgetCategory="home_screen"
    android:initialLayout="@layout/glance_default_loading_layout"
    android:updatePeriodMillis="3600000" />
```

---

## アクション処理

```kotlin
class ToggleTodoAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // Todo状態の更新
        val todoId = parameters[ActionParameters.Key<Int>("todoId")]
        // DataStore更新やDB更新
        TodoWidget().update(context, glanceId)
    }
}

// アプリを開くアクション
class OpenAppAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        val intent = Intent(context, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK
        }
        context.startActivity(intent)
    }
}
```

---

## ウィジェットの更新

```kotlin
// 外部からウィジェットを更新
class TodoRepository {
    suspend fun addTodo(todo: Todo) {
        // DB保存
        dao.insert(todo)
        // ウィジェット更新
        TodoWidget().updateAll(context)
    }
}

// WorkManagerで定期更新
class WidgetUpdateWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        TodoWidget().updateAll(applicationContext)
        return Result.success()
    }
}
```

---

## まとめ

- `GlanceAppWidget`でウィジェットUI定義
- `GlanceTheme`でMaterial3カラー対応
- `GlanceAppWidgetReceiver`でシステム連携
- `ActionCallback`でタップ/チェックボックス処理
- `updateAll()`で外部からウィジェット更新
- WorkManagerで定期更新

---

8種類のAndroidアプリテンプレート（ウィジェット対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Widget/Glanceガイド](https://zenn.dev/myougatheaxo/articles/android-widget-glance-2026)
- [WorkManagerガイド](https://zenn.dev/myougatheaxo/articles/android-work-manager-2026)
- [DataStoreガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-2026)
