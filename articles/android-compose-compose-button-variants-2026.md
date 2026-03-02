---
title: "Button完全ガイド — Button/OutlinedButton/TextButton/ElevatedButton/FilledTonalButton"
emoji: "🔲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Button**（Button、OutlinedButton、TextButton、ElevatedButton、FilledTonalButton、IconButton）を解説します。

---

## Buttonの種類

```kotlin
@Composable
fun ButtonShowcase() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // Filled Button（最も強調）
        Button(onClick = {}) { Text("Filled Button") }

        // Filled Tonal Button
        FilledTonalButton(onClick = {}) { Text("Tonal Button") }

        // Elevated Button
        ElevatedButton(onClick = {}) { Text("Elevated Button") }

        // Outlined Button
        OutlinedButton(onClick = {}) { Text("Outlined Button") }

        // Text Button（最も控えめ）
        TextButton(onClick = {}) { Text("Text Button") }
    }
}
```

---

## アイコン付きButton

```kotlin
@Composable
fun IconButtons() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // アイコン+テキスト
        Button(onClick = {}) {
            Icon(Icons.Default.Add, null, Modifier.size(18.dp))
            Spacer(Modifier.width(8.dp))
            Text("追加")
        }

        // IconButton
        IconButton(onClick = {}) {
            Icon(Icons.Default.Favorite, "お気に入り")
        }

        // Filled IconButton
        FilledIconButton(onClick = {}) {
            Icon(Icons.Default.Edit, "編集")
        }

        // FilledTonal IconButton
        FilledTonalIconButton(onClick = {}) {
            Icon(Icons.Default.Share, "共有")
        }

        // Outlined IconButton
        OutlinedIconButton(onClick = {}) {
            Icon(Icons.Default.Delete, "削除")
        }
    }
}
```

---

## ローディング付きButton

```kotlin
@Composable
fun LoadingButton(
    text: String,
    isLoading: Boolean,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Button(
        onClick = onClick,
        enabled = !isLoading,
        modifier = modifier
    ) {
        if (isLoading) {
            CircularProgressIndicator(
                modifier = Modifier.size(20.dp),
                color = MaterialTheme.colorScheme.onPrimary,
                strokeWidth = 2.dp
            )
            Spacer(Modifier.width(8.dp))
        }
        Text(if (isLoading) "処理中..." else text)
    }
}

// Toggle Button
@Composable
fun ToggleIconButton() {
    var checked by remember { mutableStateOf(false) }
    IconToggleButton(checked = checked, onCheckedChange = { checked = it }) {
        Icon(
            if (checked) Icons.Default.Favorite else Icons.Default.FavoriteBorder,
            "お気に入り",
            tint = if (checked) Color.Red else MaterialTheme.colorScheme.onSurface
        )
    }
}
```

---

## まとめ

| Button | 強調度 |
|--------|--------|
| `Button` | 最高（CTA） |
| `FilledTonalButton` | 高 |
| `ElevatedButton` | 中 |
| `OutlinedButton` | 低 |
| `TextButton` | 最低 |

- 5段階の強調度でButtonを使い分け
- `IconButton`系も5種類のバリエーション
- ローディング状態は`CircularProgressIndicator`内蔵
- `IconToggleButton`でお気に入りトグル

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Ripple/Indication](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ripple-2026)
- [ProgressIndicator](https://zenn.dev/myougatheaxo/articles/android-compose-compose-progress-indicator-2026)
- [Material Icons](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-icons-2026)
