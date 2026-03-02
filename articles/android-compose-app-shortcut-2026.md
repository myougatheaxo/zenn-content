---
title: "アプリショートカット完全ガイド — 静的/動的/ピン留め/ShortcutManager"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**アプリショートカット**（静的ショートカット、動的ショートカット、ピン留め、ShortcutManager）を解説します。

---

## 静的ショートカット

```xml
<!-- res/xml/shortcuts.xml -->
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
    <shortcut
        android:shortcutId="new_task"
        android:enabled="true"
        android:icon="@drawable/ic_add"
        android:shortcutShortLabel="@string/shortcut_new_task"
        android:shortcutLongLabel="@string/shortcut_new_task_long">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.app"
            android:targetClass="com.example.app.MainActivity"
            android:data="app://new_task" />
    </shortcut>
</shortcuts>

<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <meta-data
        android:name="android.app.shortcuts"
        android:resource="@xml/shortcuts" />
</activity>
```

---

## 動的ショートカット

```kotlin
class ShortcutHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val shortcutManager = context.getSystemService(ShortcutManager::class.java)

    fun updateRecentItems(items: List<Item>) {
        val shortcuts = items.take(4).map { item ->
            ShortcutInfoCompat.Builder(context, "item_${item.id}")
                .setShortLabel(item.title)
                .setLongLabel("${item.title}を開く")
                .setIcon(IconCompat.createWithResource(context, R.drawable.ic_item))
                .setIntent(Intent(Intent.ACTION_VIEW).apply {
                    setPackage(context.packageName)
                    data = Uri.parse("app://item/${item.id}")
                })
                .build()
        }
        ShortcutManagerCompat.setDynamicShortcuts(context, shortcuts)
    }

    fun removeShortcut(itemId: String) {
        ShortcutManagerCompat.removeDynamicShortcuts(context, listOf("item_$itemId"))
    }
}
```

---

## ピン留めショートカット

```kotlin
fun pinShortcut(item: Item) {
    if (ShortcutManagerCompat.isRequestPinShortcutSupported(context)) {
        val shortcut = ShortcutInfoCompat.Builder(context, "pin_${item.id}")
            .setShortLabel(item.title)
            .setIcon(IconCompat.createWithResource(context, R.drawable.ic_item))
            .setIntent(Intent(Intent.ACTION_VIEW).apply {
                setPackage(context.packageName)
                data = Uri.parse("app://item/${item.id}")
            })
            .build()

        ShortcutManagerCompat.requestPinShortcut(context, shortcut, null)
    }
}

// Compose UI
@Composable
fun PinShortcutButton(item: Item, shortcutHelper: ShortcutHelper) {
    OutlinedButton(onClick = { shortcutHelper.pinShortcut(item) }) {
        Icon(Icons.Default.PushPin, null)
        Spacer(Modifier.width(8.dp))
        Text("ホーム画面に追加")
    }
}
```

---

## まとめ

| 種類 | 定義場所 | 上限 |
|------|---------|------|
| 静的 | `shortcuts.xml` | 合計4個 |
| 動的 | `ShortcutManager` | 合計4個 |
| ピン留め | ユーザー操作 | 無制限 |

- 静的ショートカットでよく使う機能に即アクセス
- `ShortcutManagerCompat`で動的ショートカット管理
- `requestPinShortcut`でホーム画面にピン留め
- Deep Linkと組み合わせてNavigation連携

---

8種類のAndroidアプリテンプレート（ショートカット対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [App Links](https://zenn.dev/myougatheaxo/articles/android-compose-app-links-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [Glance Widget](https://zenn.dev/myougatheaxo/articles/android-compose-glance-widget-advanced-2026)
