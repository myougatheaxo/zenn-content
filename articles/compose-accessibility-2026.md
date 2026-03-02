---
title: "アクセシビリティ実装ガイド — Composeでa11y対応UIを作る"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

Composeでの**アクセシビリティ（a11y）対応**の実装方法を解説します。

---

## contentDescription

```kotlin
@Composable
fun AccessibleImage() {
    // 意味のある画像
    Image(
        painter = painterResource(R.drawable.product),
        contentDescription = "商品の写真：赤いスニーカー"
    )

    // 装飾画像（読み上げ不要）
    Image(
        painter = painterResource(R.drawable.decoration),
        contentDescription = null // TalkBack無視
    )

    // アイコンボタン
    IconButton(onClick = { /* 削除 */ }) {
        Icon(
            Icons.Default.Delete,
            contentDescription = "削除" // 必須
        )
    }
}
```

---

## semantics Modifier

```kotlin
@Composable
fun SemanticModifiers() {
    // カスタムアクション説明
    Box(
        Modifier
            .clickable { /* 処理 */ }
            .semantics {
                contentDescription = "設定画面を開く"
                role = Role.Button
            }
    ) {
        Text("⚙️")
    }

    // 見出しとして認識させる
    Text(
        "セクションタイトル",
        modifier = Modifier.semantics { heading() },
        style = MaterialTheme.typography.headlineMedium
    )

    // 状態の通知
    var isExpanded by remember { mutableStateOf(false) }
    Row(
        Modifier
            .clickable { isExpanded = !isExpanded }
            .semantics {
                stateDescription = if (isExpanded) "展開中" else "折りたたみ"
            }
    ) {
        Text("詳細")
        Icon(
            if (isExpanded) Icons.Default.ExpandLess else Icons.Default.ExpandMore,
            contentDescription = null
        )
    }
}
```

---

## mergeDescendants

```kotlin
@Composable
fun ProductCard(product: Product) {
    // カード全体を1つの要素として読み上げ
    Card(
        Modifier
            .fillMaxWidth()
            .semantics(mergeDescendants = true) {}
            .clickable { /* 商品詳細へ */ }
    ) {
        Column(Modifier.padding(16.dp)) {
            Text(product.name, style = MaterialTheme.typography.titleMedium)
            Text("¥${product.price}")
            Text("★ ${product.rating}")
            // TalkBack: "商品名 ¥1000 ★ 4.5"と一括読み上げ
        }
    }
}
```

---

## カスタムアクション

```kotlin
@Composable
fun SwipeableItem(
    item: Item,
    onDelete: () -> Unit,
    onArchive: () -> Unit
) {
    ListItem(
        headlineContent = { Text(item.title) },
        modifier = Modifier.semantics {
            customActions = listOf(
                CustomAccessibilityAction("削除") {
                    onDelete()
                    true
                },
                CustomAccessibilityAction("アーカイブ") {
                    onArchive()
                    true
                }
            )
        }
    )
}
```

---

## ライブリージョン

```kotlin
@Composable
fun LiveCounter() {
    var count by remember { mutableIntStateOf(0) }

    Column {
        // 値が変わるたびにTalkBackが読み上げる
        Text(
            "カウント: $count",
            modifier = Modifier.semantics {
                liveRegion = LiveRegionMode.Polite
            }
        )

        Button(onClick = { count++ }) {
            Text("増やす")
        }
    }
}
```

---

## タッチターゲット

```kotlin
@Composable
fun AccessibleButton() {
    // 最小48dp×48dpを確保
    IconButton(
        onClick = { /* 処理 */ },
        modifier = Modifier.sizeIn(minWidth = 48.dp, minHeight = 48.dp)
    ) {
        Icon(Icons.Default.Favorite, contentDescription = "お気に入り")
    }

    // 小さいチェックボックスにもタッチ領域確保
    Checkbox(
        checked = true,
        onCheckedChange = {},
        modifier = Modifier.minimumInteractiveComponentSize()
    )
}
```

---

## フォームのa11y

```kotlin
@Composable
fun AccessibleForm() {
    var email by remember { mutableStateOf("") }
    var hasError by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = email,
        onValueChange = {
            email = it
            hasError = !it.contains("@")
        },
        label = { Text("メールアドレス") },
        isError = hasError,
        supportingText = if (hasError) {
            { Text("有効なメールアドレスを入力してください") }
        } else null,
        modifier = Modifier
            .fillMaxWidth()
            .semantics {
                if (hasError) error("有効なメールアドレスを入力してください")
            }
    )
}
```

---

## まとめ

- `contentDescription`で画像・アイコンの説明
- 装飾要素は`contentDescription = null`
- `semantics { heading() }`で見出しマーク
- `mergeDescendants = true`でカード一括読み上げ
- `CustomAccessibilityAction`でスワイプ操作の代替
- `liveRegion`で動的更新の自動読み上げ
- 最小48dpタッチターゲットの確保

---

8種類のAndroidアプリテンプレート（a11y対応済み設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Modifier完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-modifier-guide-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [カスタムテーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-custom-2026)
