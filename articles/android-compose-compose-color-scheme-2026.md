---
title: "Compose ColorScheme完全ガイド — M3カラーロール/Surface/Container/Tonal"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose ColorScheme**（M3カラーロール、Surface、Container、Tonal Elevation）を解説します。

---

## カラーロール一覧

```kotlin
@Composable
fun ColorRoleDemo() {
    val cs = MaterialTheme.colorScheme
    LazyColumn(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(4.dp)) {
        item { ColorPair("primary", cs.primary, "onPrimary", cs.onPrimary) }
        item { ColorPair("primaryContainer", cs.primaryContainer, "onPrimaryContainer", cs.onPrimaryContainer) }
        item { ColorPair("secondary", cs.secondary, "onSecondary", cs.onSecondary) }
        item { ColorPair("secondaryContainer", cs.secondaryContainer, "onSecondaryContainer", cs.onSecondaryContainer) }
        item { ColorPair("tertiary", cs.tertiary, "onTertiary", cs.onTertiary) }
        item { ColorPair("error", cs.error, "onError", cs.onError) }
        item { ColorPair("surface", cs.surface, "onSurface", cs.onSurface) }
        item { ColorPair("surfaceVariant", cs.surfaceVariant, "onSurfaceVariant", cs.onSurfaceVariant) }
    }
}

@Composable
fun ColorPair(bgName: String, bg: Color, fgName: String, fg: Color) {
    Box(Modifier.fillMaxWidth().background(bg, RoundedCornerShape(8.dp)).padding(12.dp)) {
        Text("$bgName / $fgName", color = fg)
    }
}
```

---

## 実践的な使い分け

```kotlin
@Composable
fun ColorUsageExample() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // Primary: メインアクション
        Button(onClick = {}) { Text("メインアクション") }

        // SecondaryContainer: 選択状態
        FilterChip(
            selected = true,
            onClick = {},
            label = { Text("フィルター") },
            colors = FilterChipDefaults.filterChipColors(
                selectedContainerColor = MaterialTheme.colorScheme.secondaryContainer
            )
        )

        // TertiaryContainer: 補助情報
        Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.tertiaryContainer)) {
            Text("補助情報", Modifier.padding(16.dp), color = MaterialTheme.colorScheme.onTertiaryContainer)
        }

        // Error: エラー表示
        Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer)) {
            Text("エラーメッセージ", Modifier.padding(16.dp), color = MaterialTheme.colorScheme.onErrorContainer)
        }

        // SurfaceVariant: 区切り/背景
        HorizontalDivider(color = MaterialTheme.colorScheme.outlineVariant)
    }
}
```

---

## カスタム拡張カラー

```kotlin
data class ExtendedColors(
    val success: Color,
    val onSuccess: Color,
    val warning: Color,
    val onWarning: Color
)

val LocalExtendedColors = staticCompositionLocalOf {
    ExtendedColors(
        success = Color(0xFF4CAF50),
        onSuccess = Color.White,
        warning = Color(0xFFFF9800),
        onWarning = Color.White
    )
}

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    CompositionLocalProvider(LocalExtendedColors provides ExtendedColors(
        success = Color(0xFF4CAF50), onSuccess = Color.White,
        warning = Color(0xFFFF9800), onWarning = Color.White
    )) {
        MaterialTheme(content = content)
    }
}

// 使用
val extendedColors = LocalExtendedColors.current
Box(Modifier.background(extendedColors.success)) {
    Text("成功", color = extendedColors.onSuccess)
}
```

---

## まとめ

| カラーロール | 用途 |
|------------|------|
| `primary` | メインアクション/ブランド |
| `secondary` | 選択状態/フィルター |
| `tertiary` | 補助/アクセント |
| `error` | エラー表示 |
| `surface` | 背景/カード |

- M3は`primary`/`on`ペアでコントラスト保証
- `Container`系はより淡い背景色
- カスタムカラーは`CompositionLocal`で拡張
- `surfaceVariant`で微妙な背景差を表現

---

8種類のAndroidアプリテンプレート（Material3テーマ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MaterialYou](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-you-2026)
- [Compose DynamicColor](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dynamic-color-2026)
- [Compose DarkTheme](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dark-theme-2026)
