---
title: "Compose Shortcuts完全ガイド — アプリショートカット/静的/動的/ピン留め"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "shortcuts"]
published: true
---

## この記事で学べること

**Compose Shortcuts**（静的ショートカット、動的ショートカット、ピン留めショートカット）を解説します。

---

## 静的ショートカット

```xml
<!-- res/xml/shortcuts.xml -->
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
    <shortcut
        android:shortcutId="new_task"
        android:enabled="true"
        android:shortcutShortLabel="@string/shortcut_new_task"
        android:shortcutLongLabel="@string/shortcut_new_task_long"
        android:icon="@drawable/ic_add">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.app"
            android:targetClass="com.example.app.MainActivity"
            android:data="myapp://new_task" />
    </shortcut>
</shortcuts>

<!-- AndroidManifest.xml -->
<!-- <meta-data android:name="android.app.shortcuts" android:resource="@xml/shortcuts" /> -->
```

---

## 動的ショートカット

```kotlin
fun updateShortcuts(context: Context, recentItems: List<Item>) {
    val shortcuts = recentItems.take(4).map { item ->
        ShortcutInfoCompat.Builder(context, "item_${item.id}")
            .setShortLabel(item.title)
            .setLongLabel("${item.title}を開く")
            .setIcon(IconCompat.createWithResource(context, R.drawable.ic_item))
            .setIntent(Intent(Intent.ACTION_VIEW).apply {
                setPackage(context.packageName)
                data = "myapp://item/${item.id}".toUri()
            })
            .build()
    }

    ShortcutManagerCompat.setDynamicShortcuts(context, shortcuts)
}

// Compose内で更新
@Composable
fun ShortcutUpdater(viewModel: ItemViewModel = hiltViewModel()) {
    val context = LocalContext.current
    val recentItems by viewModel.recentItems.collectAsStateWithLifecycle()

    LaunchedEffect(recentItems) {
        updateShortcuts(context, recentItems)
    }
}
```

---

## ピン留めショートカット

```kotlin
fun pinShortcut(context: Context, item: Item) {
    if (ShortcutManagerCompat.isRequestPinShortcutSupported(context)) {
        val shortcut = ShortcutInfoCompat.Builder(context, "pinned_${item.id}")
            .setShortLabel(item.title)
            .setIcon(IconCompat.createWithResource(context, R.drawable.ic_item))
            .setIntent(Intent(Intent.ACTION_VIEW).apply {
                setPackage(context.packageName)
                data = "myapp://item/${item.id}".toUri()
            })
            .build()

        ShortcutManagerCompat.requestPinShortcut(context, shortcut, null)
    }
}
```

---

## まとめ

| 種別 | 定義場所 | 更新 |
|------|---------|------|
| 静的 | XML | 不可 |
| 動的 | コード | 可能 |
| ピン留め | コード | ユーザー許可 |

- 静的ショートカットはXMLで定義（最大4個）
- 動的ショートカットはコードで更新可能
- `ShortcutManagerCompat`で下位互換性確保
- DeepLinkと組み合わせてNavigation遷移

---

8種類のAndroidアプリテンプレート（ショートカット対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-deep-link-2026)
- [Compose WidgetGlance](https://zenn.dev/myougatheaxo/articles/android-compose-compose-widget-glance-2026)
- [Compose AppLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-app-link-2026)
