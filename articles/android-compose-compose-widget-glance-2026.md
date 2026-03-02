---
title: "Glance Widget完全ガイド — Compose風ウィジェット/状態管理/更新"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "widget"]
published: true
---

## この記事で学べること

**Glance Widget**（Compose風API、GlanceAppWidget、状態管理、定期更新）を解説します。

---

## Glance Widget基本

```kotlin
class TaskWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            GlanceTheme {
                TaskWidgetContent()
            }
        }
    }
}

@Composable
fun TaskWidgetContent() {
    val tasks = listOf("買い物", "レポート提出", "会議準備")

    Column(
        modifier = GlanceModifier.fillMaxSize().padding(12.dp)
            .background(GlanceTheme.colors.surface)
            .cornerRadius(16.dp)
    ) {
        Text(
            "今日のタスク",
            style = TextStyle(fontWeight = FontWeight.Bold, fontSize = 16.sp)
        )
        Spacer(GlanceModifier.height(8.dp))
        tasks.forEach { task ->
            Row(
                modifier = GlanceModifier.fillMaxWidth().padding(vertical = 4.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                CheckBox(checked = false, onCheckedChange = actionRunCallback<ToggleTaskAction>())
                Spacer(GlanceModifier.width(8.dp))
                Text(task)
            }
        }
    }
}

class TaskWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget = TaskWidget()
}
```

---

## 状態管理

```kotlin
class CounterWidget : GlanceAppWidget() {
    companion object {
        val countKey = intPreferencesKey("count")
    }

    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            val prefs = currentState<Preferences>()
            val count = prefs[countKey] ?: 0

            Column(
                modifier = GlanceModifier.fillMaxSize().padding(16.dp)
                    .background(GlanceTheme.colors.surface).cornerRadius(16.dp),
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text("$count", style = TextStyle(fontSize = 48.sp, fontWeight = FontWeight.Bold))
                Spacer(GlanceModifier.height(8.dp))
                Row {
                    Button("-", actionRunCallback<DecrementAction>())
                    Spacer(GlanceModifier.width(8.dp))
                    Button("+", actionRunCallback<IncrementAction>())
                }
            }
        }
    }
}

class IncrementAction : ActionCallback {
    override suspend fun onAction(context: Context, glanceId: GlanceId, parameters: ActionParameters) {
        updateAppWidgetState(context, glanceId) { prefs ->
            prefs[CounterWidget.countKey] = (prefs[CounterWidget.countKey] ?: 0) + 1
        }
        CounterWidget().update(context, glanceId)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GlanceAppWidget` | ウィジェット定義 |
| `provideContent` | Compose風UI |
| `ActionCallback` | インタラクション |
| `updateAppWidgetState` | 状態更新 |

- GlanceでCompose風APIによるウィジェット開発
- `ActionCallback`でユーザー操作を処理
- `Preferences`で状態を永続化
- Material3テーマに対応

---

8種類のAndroidアプリテンプレート（ウィジェット対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-chain-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
