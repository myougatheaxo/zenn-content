---
title: "App Shortcuts実装ガイド — ホーム画面から直接機能を起動"
emoji: "🔗"
type: "tech"
topics: ["android", "kotlin", "shortcuts", "ux"]
published: true
---

## この記事で学べること

Androidの**App Shortcuts**（静的・動的・ピン留め）の実装方法を解説します。

---

## 静的ショートカット

```xml
<!-- res/xml/shortcuts.xml -->
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
    <shortcut
        android:shortcutId="new_task"
        android:enabled="true"
        android:icon="@drawable/ic_add"
        android:shortcutShortLabel="@string/new_task"
        android:shortcutLongLabel="@string/create_new_task">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.app"
            android:targetClass="com.example.app.MainActivity">
            <extra android:name="shortcut_action" android:value="new_task" />
        </intent>
    </shortcut>

    <shortcut
        android:shortcutId="search"
        android:enabled="true"
        android:icon="@drawable/ic_search"
        android:shortcutShortLabel="@string/search"
        android:shortcutLongLabel="@string/search_items">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.app"
            android:targetClass="com.example.app.MainActivity">
            <extra android:name="shortcut_action" android:value="search" />
        </intent>
    </shortcut>
</shortcuts>
```

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    <meta-data
        android:name="android.app.shortcuts"
        android:resource="@xml/shortcuts" />
</activity>
```

---

## ショートカットの受け取り

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val shortcutAction = intent?.getStringExtra("shortcut_action")
            MyApp(initialAction = shortcutAction)
        }
    }
}

@Composable
fun MyApp(initialAction: String? = null) {
    val navController = rememberNavController()

    LaunchedEffect(initialAction) {
        when (initialAction) {
            "new_task" -> navController.navigate("task/new")
            "search" -> navController.navigate("search")
        }
    }

    NavHost(navController, startDestination = "home") {
        composable("home") { HomeScreen(navController) }
        composable("task/new") { NewTaskScreen() }
        composable("search") { SearchScreen() }
    }
}
```

---

## 動的ショートカット

```kotlin
class ShortcutManager(private val context: Context) {

    fun updateRecentShortcuts(recentItems: List<Item>) {
        val manager = context.getSystemService(ShortcutManager::class.java)

        val shortcuts = recentItems.take(4).map { item ->
            ShortcutInfo.Builder(context, "item_${item.id}")
                .setShortLabel(item.title)
                .setLongLabel("${item.title}を開く")
                .setIcon(Icon.createWithResource(context, R.drawable.ic_item))
                .setIntent(
                    Intent(context, MainActivity::class.java).apply {
                        action = Intent.ACTION_VIEW
                        putExtra("shortcut_action", "open_item")
                        putExtra("item_id", item.id)
                    }
                )
                .build()
        }

        manager.dynamicShortcuts = shortcuts
    }

    fun removeShortcut(itemId: String) {
        val manager = context.getSystemService(ShortcutManager::class.java)
        manager.removeDynamicShortcuts(listOf("item_$itemId"))
    }
}
```

---

## ピン留めショートカット

```kotlin
fun requestPinShortcut(context: Context, item: Item) {
    val manager = context.getSystemService(ShortcutManager::class.java)

    if (manager.isRequestPinShortcutSupported) {
        val shortcut = ShortcutInfo.Builder(context, "pin_${item.id}")
            .setShortLabel(item.title)
            .setLongLabel(item.title)
            .setIcon(Icon.createWithResource(context, R.drawable.ic_item))
            .setIntent(
                Intent(context, MainActivity::class.java).apply {
                    action = Intent.ACTION_VIEW
                    putExtra("shortcut_action", "open_item")
                    putExtra("item_id", item.id)
                }
            )
            .build()

        manager.requestPinShortcut(shortcut, null)
    }
}

// Composeから呼び出し
@Composable
fun ItemDetail(item: Item) {
    val context = LocalContext.current

    Column(Modifier.padding(16.dp)) {
        Text(item.title, style = MaterialTheme.typography.headlineMedium)

        Button(onClick = { requestPinShortcut(context, item) }) {
            Icon(Icons.Default.PushPin, null)
            Spacer(Modifier.width(8.dp))
            Text("ホーム画面に追加")
        }
    }
}
```

---

## まとめ

- **静的**: `res/xml/shortcuts.xml`で定義、変更不可
- **動的**: `ShortcutManager.dynamicShortcuts`で実行時に更新
- **ピン留め**: `requestPinShortcut()`でホーム画面に直接追加
- 最大4つまで（静的+動的合計）
- `Intent`の`extra`でアクション判定

---

8種類のAndroidアプリテンプレート（Shortcuts追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-basic-2026)
- [Widget実装ガイド](https://zenn.dev/myougatheaxo/articles/android-widget-glance-2026)
- [AlarmManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-alarm-scheduler-2026)
