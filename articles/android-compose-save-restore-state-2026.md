---
title: "状態保存/復元完全ガイド — rememberSaveable/SavedStateHandle/プロセス死"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

**状態保存/復元**（rememberSaveable、SavedStateHandle、プロセス死対応、カスタムSaver、Navigation引数）を解説します。

---

## remember vs rememberSaveable

```kotlin
@Composable
fun StateComparison() {
    // ❌ remember: 画面回転で失われる
    var count1 by remember { mutableIntStateOf(0) }

    // ✅ rememberSaveable: 画面回転・プロセス死でも保持
    var count2 by rememberSaveable { mutableIntStateOf(0) }

    Column(Modifier.padding(16.dp)) {
        Text("remember: $count1")
        Text("rememberSaveable: $count2")
        Button(onClick = { count1++; count2++ }) { Text("Increment") }
    }
}
```

---

## カスタムSaver

```kotlin
data class EditState(
    val title: String,
    val content: String,
    val isDraft: Boolean
)

val EditStateSaver = run {
    val titleKey = "title"
    val contentKey = "content"
    val isDraftKey = "isDraft"

    mapSaver(
        save = { mapOf(titleKey to it.title, contentKey to it.content, isDraftKey to it.isDraft) },
        restore = { EditState(it[titleKey] as String, it[contentKey] as String, it[isDraftKey] as Boolean) }
    )
}

@Composable
fun EditScreen() {
    var editState by rememberSaveable(stateSaver = EditStateSaver) {
        mutableStateOf(EditState("", "", true))
    }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = editState.title,
            onValueChange = { editState = editState.copy(title = it) },
            label = { Text("タイトル") }
        )
        OutlinedTextField(
            value = editState.content,
            onValueChange = { editState = editState.copy(content = it) },
            label = { Text("内容") },
            modifier = Modifier.height(200.dp)
        )
    }
}
```

---

## SavedStateHandle（ViewModel）

```kotlin
@HiltViewModel
class EditViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: ArticleRepository
) : ViewModel() {

    // SavedStateHandle で自動保存/復元
    val title = savedStateHandle.getStateFlow("title", "")
    val content = savedStateHandle.getStateFlow("content", "")

    fun updateTitle(value: String) {
        savedStateHandle["title"] = value
    }

    fun updateContent(value: String) {
        savedStateHandle["content"] = value
    }

    fun save() {
        viewModelScope.launch {
            repository.save(Article(title = title.value, content = content.value))
        }
    }
}
```

---

## リスト状態の保存

```kotlin
@Composable
fun ListWithSavedScroll() {
    val listState = rememberLazyListState() // 自動でスクロール位置保存

    LazyColumn(state = listState) {
        items(100) { index ->
            Text("Item $index", Modifier.padding(16.dp))
        }
    }
}

// 複数選択状態の保存
@Composable
fun MultiSelectList(items: List<Item>) {
    var selectedIds by rememberSaveable {
        mutableStateOf(setOf<String>())
    }

    LazyColumn {
        items(items, key = { it.id }) { item ->
            ListItem(
                headlineContent = { Text(item.title) },
                leadingContent = {
                    Checkbox(
                        checked = item.id in selectedIds,
                        onCheckedChange = { checked ->
                            selectedIds = if (checked) selectedIds + item.id
                                else selectedIds - item.id
                        }
                    )
                }
            )
        }
    }
}
```

---

## まとめ

| 手法 | スコープ | 用途 |
|------|---------|------|
| `remember` | Recomposition | 一時的な状態 |
| `rememberSaveable` | Configuration変更 | UI状態 |
| `SavedStateHandle` | プロセス死 | ViewModel状態 |
| `DataStore` | 永続 | ユーザー設定 |

- `rememberSaveable`で画面回転・プロセス死を生き延びる
- `mapSaver`/`listSaver`でカスタム型を保存
- `SavedStateHandle`でViewModelの状態を自動保存
- `LazyListState`はスクロール位置を自動保持

---

8種類のAndroidアプリテンプレート（状態管理設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [ViewModel/Hilt](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
