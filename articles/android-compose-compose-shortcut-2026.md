---
title: "Compose Shortcut完全ガイド — 静的/動的/ピン留めショートカット/DeepLink連携"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "shortcut"]
published: true
---

## この記事で学べること

**Compose Shortcut**（静的ショートカット、動的ショートカット、ピン留めショートカット、DeepLink連携）を解説します。

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
        android:shortcutLongLabel="@string/new_task_long">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="${applicationId}"
            android:targetClass=".MainActivity"
            android:data="app://tasks/new" />
    </shortcut>
</shortcuts>
```

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <meta-data android:name="android.app.shortcuts"
        android:resource="@xml/shortcuts" />
</activity>
```

---

## 動的ショートカット

```kotlin
class ShortcutManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val manager = ShortcutManagerCompat.from(context)

    fun addRecentProject(project: Project) {
        val shortcut = ShortcutInfoCompat.Builder(context, "project_${project.id}")
            .setShortLabel(project.name)
            .setLongLabel("プロジェクト: ${project.name}")
            .setIcon(IconCompat.createWithResource(context, R.drawable.ic_folder))
            .setIntent(Intent(Intent.ACTION_VIEW).apply {
                setPackage(context.packageName)
                data = Uri.parse("app://projects/${project.id}")
            })
            .setRank(0)
            .build()

        ShortcutManagerCompat.pushDynamicShortcut(context, shortcut)
    }

    fun removeShortcut(id: String) {
        ShortcutManagerCompat.removeDynamicShortcuts(context, listOf(id))
    }
}
```

---

## ピン留めショートカット

```kotlin
fun requestPinShortcut(context: Context, taskId: String, taskName: String) {
    if (!ShortcutManagerCompat.isRequestPinShortcutSupported(context)) return

    val shortcut = ShortcutInfoCompat.Builder(context, "pin_$taskId")
        .setShortLabel(taskName)
        .setIcon(IconCompat.createWithResource(context, R.drawable.ic_task))
        .setIntent(Intent(Intent.ACTION_VIEW).apply {
            setPackage(context.packageName)
            data = Uri.parse("app://tasks/$taskId")
        })
        .build()

    ShortcutManagerCompat.requestPinShortcut(context, shortcut, null)
}

@Composable
fun TaskDetailScreen(task: Task) {
    val context = LocalContext.current

    Column(Modifier.padding(16.dp)) {
        Text(task.name, style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))
        OutlinedButton(onClick = { requestPinShortcut(context, task.id, task.name) }) {
            Icon(Icons.Default.PushPin, contentDescription = null)
            Spacer(Modifier.width(8.dp))
            Text("ホームにピン留め")
        }
    }
}
```

---

## まとめ

| 種類 | 用途 |
|------|------|
| 静的ショートカット | 固定アクション |
| 動的ショートカット | 使用履歴ベース |
| ピン留め | ユーザー選択 |
| DeepLink連携 | 画面直接遷移 |

- 静的ショートカットは`shortcuts.xml`で定義
- 動的ショートカットは`pushDynamicShortcut`で最大4つ
- ピン留めショートカットでホーム画面に固定
- DeepLink URIで特定画面に直接遷移

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-deep-link-2026)
- [Compose AppLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-app-link-2026)
- [Compose Widget](https://zenn.dev/myougatheaxo/articles/android-compose-compose-widget-2026)
