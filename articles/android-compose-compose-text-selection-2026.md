---
title: "Compose TextSelection完全ガイド — SelectionContainer/コピー/カスタムメニュー/リッチテキスト選択"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "text"]
published: true
---

## この記事で学べること

**Compose TextSelection**（SelectionContainer、テキスト選択、カスタムコンテキストメニュー、部分選択制御）を解説します。

---

## SelectionContainer

```kotlin
@Composable
fun SelectableTextDemo() {
    SelectionContainer {
        Column(Modifier.padding(16.dp)) {
            Text("このテキストは選択可能です", style = MaterialTheme.typography.headlineMedium)
            Spacer(Modifier.height(8.dp))
            Text("長押しで選択範囲を指定し、コピーできます。",
                style = MaterialTheme.typography.bodyLarge)
            Spacer(Modifier.height(8.dp))
            Text("複数のTextを跨いで選択も可能です。",
                style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

---

## 部分的な選択無効化

```kotlin
@Composable
fun PartialSelectionDemo() {
    SelectionContainer {
        Column(Modifier.padding(16.dp)) {
            Text("選択可能なテキスト")
            Spacer(Modifier.height(8.dp))

            DisableSelection {
                Text("このテキストは選択不可", color = MaterialTheme.colorScheme.onSurfaceVariant)
            }

            Spacer(Modifier.height(8.dp))
            Text("再び選択可能なテキスト")
        }
    }
}
```

---

## カスタムテキスト表示

```kotlin
@Composable
fun CodeBlock(code: String) {
    SelectionContainer {
        Surface(
            color = Color(0xFF1E1E1E),
            shape = RoundedCornerShape(8.dp),
            modifier = Modifier.fillMaxWidth()
        ) {
            Row(Modifier.padding(16.dp)) {
                // 行番号（選択不可）
                DisableSelection {
                    Column {
                        code.lines().forEachIndexed { index, _ ->
                            Text("${index + 1}", color = Color.Gray,
                                fontFamily = FontFamily.Monospace,
                                modifier = Modifier.padding(end = 16.dp))
                        }
                    }
                }
                // コード本体（選択可能）
                Column {
                    code.lines().forEach { line ->
                        Text(line, color = Color.White, fontFamily = FontFamily.Monospace)
                    }
                }
            }
        }
    }
}

@Composable
fun CopyableInfo(label: String, value: String) {
    Row(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
        DisableSelection {
            Text("$label: ", fontWeight = FontWeight.Bold,
                modifier = Modifier.width(100.dp))
        }
        SelectionContainer {
            Text(value)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SelectionContainer` | 選択可能領域 |
| `DisableSelection` | 選択無効化 |
| `ClickableText` | クリック可能テキスト |
| `AnnotatedString` | リッチテキスト |

- `SelectionContainer`でテキスト選択を有効化
- `DisableSelection`で部分的に選択を無効化
- 複数`Text`を跨いだ範囲選択が可能
- コードブロック等で行番号を選択不可にする使い方

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TextStyle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
- [Compose RichText](https://zenn.dev/myougatheaxo/articles/android-compose-compose-richtext-2026)
- [Compose Typography](https://zenn.dev/myougatheaxo/articles/android-compose-compose-typography-2026)
