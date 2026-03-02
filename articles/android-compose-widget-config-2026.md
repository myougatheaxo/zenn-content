---
title: "ウィジェット設定完全ガイド — GlanceAppWidgetReceiver/設定Activity/更新間隔"
emoji: "⚙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "widget"]
published: true
---

## この記事で学べること

**ウィジェット設定**（GlanceAppWidgetReceiver、設定Activity、更新間隔、データ保存）を解説します。

---

## 設定Activity

```kotlin
class WidgetConfigActivity : ComponentActivity() {
    private var appWidgetId = AppWidgetManager.INVALID_APPWIDGET_ID

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        appWidgetId = intent.extras?.getInt(
            AppWidgetManager.EXTRA_APPWIDGET_ID,
            AppWidgetManager.INVALID_APPWIDGET_ID
        ) ?: AppWidgetManager.INVALID_APPWIDGET_ID

        setResult(RESULT_CANCELED)

        setContent {
            MaterialTheme {
                WidgetConfigScreen(
                    onSave = { config -> saveAndFinish(config) }
                )
            }
        }
    }

    private fun saveAndFinish(config: WidgetConfig) {
        // 設定保存
        val prefs = getSharedPreferences("widget_prefs", MODE_PRIVATE)
        prefs.edit()
            .putString("title_$appWidgetId", config.title)
            .putInt("color_$appWidgetId", config.color.toArgb())
            .apply()

        // ウィジェット更新
        val appWidgetManager = AppWidgetManager.getInstance(this)
        MyWidget().update(this, appWidgetManager, appWidgetId)

        setResult(RESULT_OK, Intent().putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetId))
        finish()
    }
}

@Composable
fun WidgetConfigScreen(onSave: (WidgetConfig) -> Unit) {
    var title by remember { mutableStateOf("") }
    var selectedColor by remember { mutableStateOf(Color(0xFF6200EE)) }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("ウィジェット設定", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(16.dp))

        OutlinedTextField(
            value = title,
            onValueChange = { title = it },
            label = { Text("タイトル") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(Modifier.height(16.dp))
        Text("テーマカラー")

        val colors = listOf(Color(0xFF6200EE), Color(0xFF03DAC5), Color(0xFFFF5722), Color(0xFF4CAF50))
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            colors.forEach { color ->
                Box(
                    Modifier
                        .size(48.dp)
                        .clip(CircleShape)
                        .background(color)
                        .border(if (selectedColor == color) 3.dp else 0.dp, Color.Black, CircleShape)
                        .clickable { selectedColor = color }
                )
            }
        }

        Spacer(Modifier.weight(1f))
        Button(
            onClick = { onSave(WidgetConfig(title, selectedColor)) },
            modifier = Modifier.fillMaxWidth()
        ) { Text("保存") }
    }
}
```

---

## Manifest宣言

```xml
<receiver android:name=".MyWidgetReceiver" android:exported="false">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/widget_info" />
</receiver>

<activity android:name=".WidgetConfigActivity" android:exported="false">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_CONFIGURE" />
    </intent-filter>
</activity>
```

```xml
<!-- res/xml/widget_info.xml -->
<appwidget-provider
    android:configure="com.example.app.WidgetConfigActivity"
    android:initialLayout="@layout/widget_loading"
    android:minWidth="180dp"
    android:minHeight="110dp"
    android:updatePeriodMillis="1800000"
    android:resizeMode="horizontal|vertical" />
```

---

## まとめ

| 設定項目 | 場所 |
|---------|------|
| 設定Activity | `android:configure` |
| 更新間隔 | `updatePeriodMillis` |
| サイズ | `minWidth`/`minHeight` |
| リサイズ | `resizeMode` |

- 設定Activityでユーザーカスタマイズ
- `SharedPreferences`でウィジェット個別設定保存
- `updatePeriodMillis`で自動更新間隔設定
- `resizeMode`でリサイズ方向を制御

---

8種類のAndroidアプリテンプレート（ウィジェット対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Glance Widget](https://zenn.dev/myougatheaxo/articles/android-compose-glance-widget-advanced-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
