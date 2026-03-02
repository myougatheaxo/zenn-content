---
title: "Compose RTL完全ガイド — 右から左レイアウト/アラビア語対応/ミラーリング/CompositionLocal"
emoji: "↩️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "rtl"]
published: true
---

## この記事で学べること

**Compose RTL**（右から左レイアウト、アラビア語/ヘブライ語対応、アイコンミラーリング、レイアウト方向制御）を解説します。

---

## RTL自動対応

```kotlin
@Composable
fun RtlAwareLayout() {
    // Composeは自動的にRTLに対応
    Row(
        Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "戻る")
        Text("タイトル", style = MaterialTheme.typography.titleLarge)
        Icon(Icons.Default.MoreVert, contentDescription = "メニュー")
    }
}

// start/endを使用（left/rightではなく）
@Composable
fun RtlSafeCard() {
    Card(Modifier.fillMaxWidth().padding(16.dp)) {
        Row(Modifier.padding(16.dp)) {
            Icon(Icons.Default.Person, contentDescription = null)
            Spacer(Modifier.width(16.dp))
            Column(Modifier.weight(1f)) {
                Text("ユーザー名", style = MaterialTheme.typography.titleMedium)
                Text("説明テキスト")
            }
            // trailingアイコンはRTLで左側に移動
            Icon(Icons.AutoMirrored.Filled.KeyboardArrowRight, contentDescription = null)
        }
    }
}
```

---

## レイアウト方向制御

```kotlin
@Composable
fun ForceDirectionDemo() {
    Column(Modifier.padding(16.dp)) {
        // 強制LTR（数値、電話番号等）
        CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Ltr) {
            Text("+81-90-1234-5678", style = MaterialTheme.typography.bodyLarge)
        }

        Spacer(Modifier.height(16.dp))

        // 強制RTL（テスト用）
        CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Rtl) {
            Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                Text("右")
                Text("左")
            }
        }
    }
}
```

---

## RTLテスト

```kotlin
@Composable
fun RtlPreview() {
    Column {
        // LTR版
        Text("LTR Layout:", fontWeight = FontWeight.Bold)
        CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Ltr) {
            SampleListItem()
        }

        Spacer(Modifier.height(16.dp))

        // RTL版
        Text("RTL Layout:", fontWeight = FontWeight.Bold)
        CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Rtl) {
            SampleListItem()
        }
    }
}

@Composable
fun SampleListItem() {
    ListItem(
        headlineContent = { Text("List Item") },
        leadingContent = { Icon(Icons.Default.Star, null) },
        trailingContent = { Icon(Icons.AutoMirrored.Filled.KeyboardArrowRight, null) }
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LocalLayoutDirection` | 方向制御 |
| `Arrangement.Start/End` | RTL対応配置 |
| `AutoMirrored` | アイコンミラー |
| `start/end padding` | RTL対応余白 |

- Composeは`Row`/`Column`が自動的にRTL対応
- `start`/`end`を`left`/`right`の代わりに使用
- `Icons.AutoMirrored`でRTL時にアイコンをミラーリング
- `CompositionLocalProvider`でレイアウト方向を制御

---

8種類のAndroidアプリテンプレート（多言語対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Localization](https://zenn.dev/myougatheaxo/articles/android-compose-compose-localization-2026)
- [Compose Accessibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-accessibility-2026)
- [Compose AdaptiveLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
