---
title: "Composeアクセシビリティ実践ガイド — TalkBack/セマンティクス対応"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

Composeの**アクセシビリティ対応**（セマンティクス、TalkBack、コンテンツ説明）を解説します。

---

## セマンティクスの基本

```kotlin
@Composable
fun AccessibleImage() {
    // ✅ 装飾画像（読み上げ不要）
    Image(
        painter = painterResource(R.drawable.decorative),
        contentDescription = null // TalkBackでスキップ
    )

    // ✅ 意味のある画像
    Image(
        painter = painterResource(R.drawable.profile),
        contentDescription = "プロフィール画像"
    )
}
```

---

## セマンティクスのマージ

```kotlin
@Composable
fun AccessibleListItem(item: Item, onClick: () -> Unit) {
    // ❌ 各要素が個別に読み上げられる
    Row(Modifier.clickable { onClick() }) {
        Text(item.title)
        Text(item.subtitle)
    }

    // ✅ 1つの要素として読み上げ
    Row(
        Modifier
            .semantics(mergeDescendants = true) {}
            .clickable { onClick() }
    ) {
        Text(item.title)
        Text(item.subtitle)
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
    Box(
        Modifier.semantics {
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
    ) {
        Text(item.title)
    }
}
```

---

## 状態の読み上げ

```kotlin
@Composable
fun AccessibleToggle() {
    var isEnabled by remember { mutableStateOf(false) }

    Row(
        Modifier
            .semantics(mergeDescendants = true) {}
            .toggleable(
                value = isEnabled,
                role = Role.Switch,
                onValueChange = { isEnabled = it }
            )
            .padding(16.dp)
    ) {
        Text("通知設定", Modifier.weight(1f))
        Switch(
            checked = isEnabled,
            onCheckedChange = null // Rowで処理
        )
    }
}

@Composable
fun AccessibleSlider() {
    var volume by remember { mutableFloatStateOf(0.5f) }

    Column(Modifier.semantics(mergeDescendants = true) {}) {
        Text("音量: ${(volume * 100).toInt()}%")
        Slider(
            value = volume,
            onValueChange = { volume = it },
            modifier = Modifier.semantics {
                stateDescription = "音量 ${(volume * 100).toInt()}パーセント"
            }
        )
    }
}
```

---

## ライブリージョン

```kotlin
@Composable
fun LiveRegionExample() {
    var count by remember { mutableIntStateOf(0) }

    Column {
        Button(onClick = { count++ }) {
            Text("カウントアップ")
        }

        // ✅ 値変更時にTalkBackが自動読み上げ
        Text(
            "カウント: $count",
            modifier = Modifier.semantics {
                liveRegion = LiveRegionMode.Polite
            }
        )
    }
}
```

---

## まとめ

- `contentDescription`で画像の説明を提供
- `mergeDescendants`で関連要素をグループ化
- `customActions`でスワイプ等のアクションを代替提供
- `Role`で要素の役割をTalkBackに伝達
- `liveRegion`で動的変更を自動読み上げ
- テスト時はTalkBackを有効にして確認

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムレイアウトガイド](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
- [Material3テーマガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
- [テスト実践ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-testing-screenshot-2026)
