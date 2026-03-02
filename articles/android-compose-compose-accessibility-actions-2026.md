---
title: "Accessibility Actions完全ガイド — カスタムアクション/contentDescription/TalkBack"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

**Accessibility Actions**（カスタムアクション、contentDescription、TalkBack対応、semantics）を解説します。

---

## contentDescription

```kotlin
@Composable
fun AccessibleUI() {
    Column(Modifier.padding(16.dp)) {
        // 画像にcontentDescription
        Image(
            painter = painterResource(R.drawable.product),
            contentDescription = "商品画像: ワイヤレスイヤホン",
            modifier = Modifier.size(200.dp)
        )

        // 装飾的な画像はnull
        Icon(Icons.Default.Star, contentDescription = null)

        // ボタンのアクセシビリティ
        IconButton(
            onClick = {},
            modifier = Modifier.semantics { contentDescription = "お気に入りに追加" }
        ) {
            Icon(Icons.Default.FavoriteBorder, null)
        }
    }
}
```

---

## カスタムアクション

```kotlin
@Composable
fun SwipeableCard(item: String, onDelete: () -> Unit, onArchive: () -> Unit) {
    Card(
        Modifier
            .fillMaxWidth()
            .semantics {
                customActions = listOf(
                    CustomAccessibilityAction("削除") { onDelete(); true },
                    CustomAccessibilityAction("アーカイブ") { onArchive(); true }
                )
            }
    ) {
        ListItem(
            headlineContent = { Text(item) },
            trailingContent = {
                Row {
                    IconButton(onClick = onArchive) {
                        Icon(Icons.Default.Archive, contentDescription = "アーカイブ")
                    }
                    IconButton(onClick = onDelete) {
                        Icon(Icons.Default.Delete, contentDescription = "削除")
                    }
                }
            }
        )
    }
}
```

---

## 状態の読み上げ

```kotlin
@Composable
fun AccessibleCounter() {
    var count by remember { mutableIntStateOf(0) }

    Row(
        Modifier
            .semantics(mergeDescendants = true) {
                stateDescription = "${count}個選択中"
            }
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        IconButton(onClick = { if (count > 0) count-- }) {
            Icon(Icons.Default.Remove, "減らす")
        }
        Text(
            "$count",
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.padding(horizontal = 16.dp)
                .semantics { liveRegion = LiveRegionMode.Polite }
        )
        IconButton(onClick = { count++ }) {
            Icon(Icons.Default.Add, "増やす")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `contentDescription` | 読み上げテキスト |
| `customActions` | カスタムアクション |
| `stateDescription` | 状態説明 |
| `liveRegion` | 変更の自動通知 |

- `contentDescription`で画像/アイコンの説明
- `customActions`でスワイプ等の代替操作
- `mergeDescendants`で関連要素をグループ化
- `liveRegion`で動的変更をTalkBackに通知

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Semantics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-semantics-2026)
- [TestTag](https://zenn.dev/myougatheaxo/articles/android-compose-compose-test-tag-2026)
- [UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ui-test-2026)
