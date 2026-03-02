---
title: "Divider完全ガイド — HorizontalDivider/VerticalDivider/カスタム区切り線"
emoji: "➖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Divider**（HorizontalDivider、VerticalDivider、カスタム区切り線、インセット）を解説します。

---

## HorizontalDivider

```kotlin
@Composable
fun DividerExamples() {
    Column(Modifier.padding(16.dp)) {
        Text("セクション1", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))
        Text("コンテンツ1")

        HorizontalDivider(Modifier.padding(vertical = 16.dp))

        Text("セクション2", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))
        Text("コンテンツ2")

        // カスタムDivider
        HorizontalDivider(
            modifier = Modifier.padding(vertical = 16.dp),
            thickness = 2.dp,
            color = MaterialTheme.colorScheme.primary
        )

        Text("セクション3")
    }
}
```

---

## VerticalDivider

```kotlin
@Composable
fun VerticalDividerExample() {
    Row(
        Modifier.height(IntrinsicSize.Min).padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Column(Modifier.weight(1f)) {
            Text("左カラム", fontWeight = FontWeight.Bold)
            Text("左のコンテンツ")
        }

        VerticalDivider(
            modifier = Modifier.fillMaxHeight(),
            thickness = 1.dp,
            color = MaterialTheme.colorScheme.outline
        )

        Column(Modifier.weight(1f)) {
            Text("右カラム", fontWeight = FontWeight.Bold)
            Text("右のコンテンツ")
        }
    }
}
```

---

## リスト内Divider

```kotlin
@Composable
fun ListWithDividers(items: List<String>) {
    LazyColumn {
        itemsIndexed(items) { index, item ->
            ListItem(
                headlineContent = { Text(item) },
                leadingContent = { Icon(Icons.Default.Article, null) }
            )
            if (index < items.lastIndex) {
                HorizontalDivider(
                    modifier = Modifier.padding(start = 56.dp), // インセット
                    color = MaterialTheme.colorScheme.outlineVariant
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `HorizontalDivider` | 水平区切り線 |
| `VerticalDivider` | 垂直区切り線 |
| `thickness` | 線の太さ |
| `color` | 線の色 |

- `HorizontalDivider`でセクション間の区切り
- `VerticalDivider`でカラム間の区切り
- `padding(start = 56.dp)`でインセット付きDivider
- `outlineVariant`色で目立ちすぎない区切り

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Surface](https://zenn.dev/myougatheaxo/articles/android-compose-compose-surface-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
