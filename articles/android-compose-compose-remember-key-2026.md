---
title: "remember/key完全ガイド — remember/rememberSaveable/key指定/カスタムSaver"
emoji: "🔑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

**remember/key**（remember、rememberSaveable、key指定による再計算、カスタムSaver）を解説します。

---

## rememberとkey

```kotlin
@Composable
fun RememberKeyExample(userId: Long) {
    // userIdが変わると再計算
    val userProfile = remember(userId) {
        expensiveCalculation(userId)
    }

    // 複数キー
    val filtered = remember(userId, query) {
        allItems.filter { it.userId == userId && it.name.contains(query) }
    }

    // キーなし（初回のみ計算、リコンポーズで保持）
    val interactionSource = remember { MutableInteractionSource() }
}

@Composable
fun AnimatedCounter(targetCount: Int) {
    val animatedCount by animateIntAsState(
        targetValue = targetCount,
        animationSpec = tween(500),
        label = "count"
    )
    Text("$animatedCount", fontSize = 48.sp)
}
```

---

## rememberSaveable

```kotlin
@Composable
fun SaveableExample() {
    // プロセス再生成でも復元（Bundle保存可能な型）
    var text by rememberSaveable { mutableStateOf("") }
    var count by rememberSaveable { mutableIntStateOf(0) }

    // カスタムSaver
    var selectedDate by rememberSaveable(stateSaver = DateSaver) {
        mutableStateOf(LocalDate.now())
    }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(value = text, onValueChange = { text = it })
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("増やす") }
    }
}

val DateSaver = Saver<LocalDate, String>(
    save = { it.toString() },
    restore = { LocalDate.parse(it) }
)
```

---

## リストのカスタムSaver

```kotlin
data class TodoItem(val id: Long, val title: String, val done: Boolean)

val TodoListSaver = listSaver<List<TodoItem>, Any>(
    save = { list ->
        list.flatMap { listOf(it.id, it.title, it.done) }
    },
    restore = { values ->
        values.chunked(3).map { (id, title, done) ->
            TodoItem(id as Long, title as String, done as Boolean)
        }
    }
)

@Composable
fun TodoScreen() {
    var todos by rememberSaveable(stateSaver = TodoListSaver) {
        mutableStateOf(emptyList())
    }

    LazyColumn {
        items(todos, key = { it.id }) { todo ->
            ListItem(
                headlineContent = { Text(todo.title) },
                trailingContent = {
                    Checkbox(checked = todo.done, onCheckedChange = { checked ->
                        todos = todos.map {
                            if (it.id == todo.id) it.copy(done = checked) else it
                        }
                    })
                }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `remember` | リコンポーズ間で値保持 |
| `remember(key)` | keyが変わると再計算 |
| `rememberSaveable` | プロセス再生成後も復元 |
| `Saver` | カスタム型のBundle保存 |

- `remember(key)`で依存値変更時に自動再計算
- `rememberSaveable`で画面回転・プロセスキルに対応
- カスタム`Saver`で任意の型をBundle保存可能
- `listSaver`/`mapSaver`でコレクション保存

---

8種類のAndroidアプリテンプレート（状態管理実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State Management](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [State Hoisting](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-hoisting-2026)
- [Recomposition](https://zenn.dev/myougatheaxo/articles/android-compose-compose-recomposition-2026)
