---
title: "アクセシビリティ基礎ガイド — Semantics/contentDescription/heading"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

**Composeアクセシビリティ**（Semantics、contentDescription、heading、フォーカス順序、タッチターゲット）を解説します。

---

## contentDescription

```kotlin
// 情報のある画像
Image(
    painter = painterResource(R.drawable.profile),
    contentDescription = "ユーザーのプロフィール写真"
)

// 装飾画像（読み上げ不要）
Image(
    painter = painterResource(R.drawable.background),
    contentDescription = null // TalkBackが無視
)

// アイコンボタン
IconButton(onClick = { /* 検索 */ }) {
    Icon(
        Icons.Default.Search,
        contentDescription = "検索" // ボタンの用途を説明
    )
}
```

---

## Semantics

```kotlin
// カスタムSemanticsの追加
@Composable
fun PriceTag(price: Int) {
    Text(
        text = "¥${"%,d".format(price)}",
        modifier = Modifier.semantics {
            contentDescription = "${price}円"
        }
    )
}

// 見出し指定
@Composable
fun SectionHeader(title: String) {
    Text(
        text = title,
        style = MaterialTheme.typography.headlineMedium,
        modifier = Modifier.semantics { heading() }
    )
}

// 状態の説明
@Composable
fun StatusBadge(isOnline: Boolean) {
    Box(
        modifier = Modifier
            .size(12.dp)
            .background(if (isOnline) Color.Green else Color.Gray, CircleShape)
            .semantics {
                contentDescription = if (isOnline) "オンライン" else "オフライン"
                stateDescription = if (isOnline) "接続中" else "未接続"
            }
    )
}
```

---

## mergeDescendants

```kotlin
// 子要素を1つのノードとして読み上げ
@Composable
fun ContactItem(contact: Contact) {
    Row(
        modifier = Modifier
            .semantics(mergeDescendants = true) {}
            .clickable { /* タップ処理 */ }
            .padding(16.dp)
    ) {
        AsyncImage(
            model = contact.avatarUrl,
            contentDescription = null // 親にマージされるので不要
        )
        Spacer(Modifier.width(12.dp))
        Column {
            Text(contact.name)
            Text(contact.lastMessage)
        }
    }
    // TalkBack: "田中太郎 今日は暇？" と一括読み上げ
}
```

---

## タッチターゲット

```kotlin
// 最小48dp確保
@Composable
fun SmallButton(onClick: () -> Unit) {
    IconButton(
        onClick = onClick,
        modifier = Modifier.sizeIn(minWidth = 48.dp, minHeight = 48.dp)
    ) {
        Icon(
            Icons.Default.Close,
            contentDescription = "閉じる",
            modifier = Modifier.size(24.dp) // 見た目は小さくてもOK
        )
    }
}

// Material3は自動で48dp以上を確保
// ただし手動でsizeを小さくするとタッチ困難に
```

---

## フォーカス順序

```kotlin
@Composable
fun LoginForm() {
    val (emailFocus, passwordFocus, buttonFocus) = remember { FocusRequester.createRefs() }

    Column {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メールアドレス") },
            modifier = Modifier
                .focusRequester(emailFocus)
                .focusProperties { next = passwordFocus },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(onNext = { passwordFocus.requestFocus() })
        )

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            modifier = Modifier
                .focusRequester(passwordFocus)
                .focusProperties { next = buttonFocus },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            keyboardActions = KeyboardActions(onDone = { buttonFocus.requestFocus() })
        )

        Button(
            onClick = { login() },
            modifier = Modifier.focusRequester(buttonFocus)
        ) {
            Text("ログイン")
        }
    }
}
```

---

## liveRegion

```kotlin
// 動的に変わるテキストを自動読み上げ
@Composable
fun ErrorMessage(message: String?) {
    message?.let {
        Text(
            text = it,
            color = MaterialTheme.colorScheme.error,
            modifier = Modifier.semantics {
                liveRegion = LiveRegionMode.Polite
            }
        )
    }
}

// カウンターの変化を通知
@Composable
fun CartBadge(count: Int) {
    Badge(
        modifier = Modifier.semantics {
            contentDescription = "カート: ${count}個"
            liveRegion = LiveRegionMode.Polite
        }
    ) {
        Text("$count")
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 画像説明 | `contentDescription` |
| 見出し | `semantics { heading() }` |
| グループ化 | `mergeDescendants = true` |
| 動的通知 | `liveRegion` |
| タッチサイズ | `sizeIn(48.dp)` |
| フォーカス | `FocusRequester` |

- 装飾画像は`contentDescription = null`
- `mergeDescendants`で論理的なグループ読み上げ
- `liveRegion`で動的変化を通知
- タッチターゲット48dp以上確保

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アクセシビリティテスト](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-testing-2026)
- [ComposeTestRule](https://zenn.dev/myougatheaxo/articles/android-compose-testing-compose-rule-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
