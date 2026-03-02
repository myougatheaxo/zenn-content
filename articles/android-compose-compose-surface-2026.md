---
title: "Surface完全ガイド — elevation/shape/tonalElevation/clickable"
emoji: "🎴"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Surface**（elevation、shape、tonalElevation、clickable Surface）を解説します。

---

## 基本Surface

```kotlin
@Composable
fun SurfaceExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // デフォルト
        Surface(tonalElevation = 1.dp) {
            Text("Tonal Elevation 1dp", Modifier.padding(16.dp))
        }

        // 影付き
        Surface(
            shadowElevation = 4.dp,
            shape = RoundedCornerShape(12.dp)
        ) {
            Text("Shadow Elevation 4dp", Modifier.padding(16.dp))
        }

        // カラー指定
        Surface(
            color = MaterialTheme.colorScheme.primaryContainer,
            contentColor = MaterialTheme.colorScheme.onPrimaryContainer,
            shape = RoundedCornerShape(12.dp)
        ) {
            Text("Primary Container", Modifier.padding(16.dp))
        }
    }
}
```

---

## Clickable Surface

```kotlin
@Composable
fun ClickableSurface(title: String, onClick: () -> Unit) {
    Surface(
        onClick = onClick,
        shape = RoundedCornerShape(12.dp),
        tonalElevation = 2.dp,
        modifier = Modifier.fillMaxWidth()
    ) {
        Row(
            Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text(title, Modifier.weight(1f), style = MaterialTheme.typography.titleMedium)
            Icon(Icons.Default.ChevronRight, null)
        }
    }
}

// 設定画面
@Composable
fun SettingsScreen() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        listOf("アカウント", "通知", "テーマ", "言語", "プライバシー").forEach { item ->
            ClickableSurface(title = item, onClick = {})
        }
    }
}
```

---

## Tonal Elevation

```kotlin
@Composable
fun TonalElevationShowcase() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        listOf(0.dp, 1.dp, 3.dp, 6.dp, 8.dp, 12.dp).forEach { elevation ->
            Surface(
                tonalElevation = elevation,
                shape = RoundedCornerShape(8.dp),
                modifier = Modifier.fillMaxWidth()
            ) {
                Text(
                    "Tonal Elevation: $elevation",
                    Modifier.padding(16.dp)
                )
            }
        }
    }
}
```

---

## まとめ

| パラメータ | 用途 |
|----------|------|
| `tonalElevation` | 色調変化 |
| `shadowElevation` | 影 |
| `shape` | 形状 |
| `onClick` | クリック対応 |

- `tonalElevation`でMaterial3の色調変化
- `shadowElevation`でドロップシャドウ
- `Surface(onClick)`でリップル付きクリック
- `color`/`contentColor`でカスタムカラー

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [Border/Shadow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-border-2026)
- [ダイナミックテーマ](https://zenn.dev/myougatheaxo/articles/android-compose-dynamic-theme-2026)
