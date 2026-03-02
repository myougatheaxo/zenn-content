---
title: "ボタン全種類ガイド — Button/IconButton/SegmentedButtonの使い分け"
emoji: "🔘"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "button"]
published: true
---

## この記事で学べること

ComposeのMaterial 3**ボタン全種類**の使い分けと実装方法を解説します。

---

## ボタンの種類と使い分け

```kotlin
@Composable
fun ButtonVariants() {
    Column(
        Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        // Filled Button（最重要アクション）
        Button(onClick = { }) {
            Text("保存する")
        }

        // Filled Tonal（重要だが目立ちすぎない）
        FilledTonalButton(onClick = { }) {
            Text("下書き保存")
        }

        // Outlined（代替アクション）
        OutlinedButton(onClick = { }) {
            Text("キャンセル")
        }

        // Text Button（最低優先度）
        TextButton(onClick = { }) {
            Text("スキップ")
        }

        // Elevated（影付き）
        ElevatedButton(onClick = { }) {
            Text("詳細を見る")
        }
    }
}
```

---

## アイコン付きボタン

```kotlin
@Composable
fun IconButtons() {
    Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // アイコン+テキスト
        Button(onClick = { }) {
            Icon(Icons.Default.Add, null, Modifier.size(18.dp))
            Spacer(Modifier.width(8.dp))
            Text("新規作成")
        }

        // IconButton（アイコンのみ）
        IconButton(onClick = { }) {
            Icon(Icons.Default.Favorite, "お気に入り")
        }

        // Filled IconButton
        FilledIconButton(onClick = { }) {
            Icon(Icons.Default.Edit, "編集")
        }

        // Outlined IconButton
        OutlinedIconButton(onClick = { }) {
            Icon(Icons.Default.Share, "共有")
        }

        // Toggle IconButton
        var toggled by remember { mutableStateOf(false) }
        FilledIconToggleButton(
            checked = toggled,
            onCheckedChange = { toggled = it }
        ) {
            Icon(
                if (toggled) Icons.Default.Favorite else Icons.Default.FavoriteBorder,
                "お気に入り"
            )
        }
    }
}
```

---

## ローディングボタン

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

// 使用例
@Composable
fun SubmitForm() {
    var isLoading by remember { mutableStateOf(false) }

    LoadingButton(
        text = "送信",
        isLoading = isLoading,
        onClick = {
            isLoading = true
            // 処理後にisLoading = false
        },
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## カスタムスタイル

```kotlin
@Composable
fun StyledButtons() {
    // カスタムカラー
    Button(
        onClick = { },
        colors = ButtonDefaults.buttonColors(
            containerColor = Color(0xFF4CAF50),
            contentColor = Color.White
        )
    ) {
        Text("カスタム色")
    }

    // カスタム形状
    Button(
        onClick = { },
        shape = RoundedCornerShape(50)
    ) {
        Text("丸角ボタン")
    }

    // フルワイドボタン
    Button(
        onClick = { },
        modifier = Modifier
            .fillMaxWidth()
            .height(56.dp),
        shape = RoundedCornerShape(12.dp)
    ) {
        Text("フルワイドボタン", fontSize = 16.sp)
    }

    // グラデーションボタン
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(56.dp)
            .clip(RoundedCornerShape(12.dp))
            .background(
                Brush.horizontalGradient(
                    listOf(Color(0xFF6200EE), Color(0xFF03DAC6))
                )
            )
            .clickable { /* 処理 */ },
        contentAlignment = Alignment.Center
    ) {
        Text("グラデーション", color = Color.White, fontWeight = FontWeight.Bold)
    }
}
```

---

## ボタングループ

```kotlin
@Composable
fun ButtonGroup() {
    // 横並び
    Row(
        Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        OutlinedButton(
            onClick = { },
            modifier = Modifier.weight(1f)
        ) {
            Text("キャンセル")
        }
        Button(
            onClick = { },
            modifier = Modifier.weight(1f)
        ) {
            Text("確定")
        }
    }

    Spacer(Modifier.height(16.dp))

    // 縦並び
    Column(
        Modifier.fillMaxWidth(),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        Button(onClick = { }, Modifier.fillMaxWidth()) { Text("メインアクション") }
        OutlinedButton(onClick = { }, Modifier.fillMaxWidth()) { Text("サブアクション") }
        TextButton(onClick = { }, Modifier.fillMaxWidth()) { Text("スキップ") }
    }
}
```

---

## まとめ

| ボタン | 用途 |
|--------|------|
| `Button` | 最重要アクション（1画面に1つ推奨） |
| `FilledTonalButton` | 重要だが控えめ |
| `OutlinedButton` | 代替アクション |
| `TextButton` | 最低優先度（キャンセル等） |
| `ElevatedButton` | Surface上で目立たせたい場合 |
| `IconButton` | アイコンのみのアクション |
| `FilledIconToggleButton` | ON/OFF切り替え |

---

8種類のAndroidアプリテンプレート（ボタンUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [FAB実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-floating-action-button-2026)
- [Material3コンポーネント一覧](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [Modifier完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-modifier-guide-2026)
